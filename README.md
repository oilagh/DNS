# 4700dns — DNS Resolver

## High-Level Approach

The program is a DNS server written in Python that does two things at once: it serves as an authoritative nameserver for a local zone file, and it acts as a full recursive resolver for everything outside that zone. When a client sends it a query, it figures out which mode applies and handles it accordingly. If the name being asked about lives inside the loaded zone file, the server answers directly from its own records. If not, it goes out and hunts down the answer by walking the DNS hierarchy from the root, following referrals one nameserver at a time until it lands somewhere that actually knows the answer.

The whole thing runs over UDP. Each incoming datagram gets handed off to its own thread so multiple clients can be handled at the same time without one slow resolution blocking everyone else. The only shared state is the cache, which is protected by a lock so threads don't step on each other.

Defensiveness was a core design principle throughout. The server never trusts data from the outside world. Every response from an upstream nameserver gets filtered before anything in it is believed or stored. Malformed packets, unexpected response codes, servers that go silent, servers that lie, truncated responses, infinite CNAME loops — all of these are handled without the server crashing, hanging, or returning garbage to the client. The worst outcome for any query is a SERVFAIL, never a crash or a corrupt answer.

## Authoritative Mode

When a query comes in for a name that falls inside the server's own zone, `handle_authoritative` takes over. The first thing it does is check whether the name actually belongs to a sub-zone that has been delegated out. It does this by walking up the labels of the queried name one step at a time and checking whether any ancestor has NS records in the zone data. If it finds one — say, the query is for `www.sub.example.com` and `sub.example.com` has NS records — it returns a referral instead of trying to answer. The referral puts the NS records in the authority section and, if the server knows the IP addresses of those nameservers (glue records), includes them in the additional section. This is exactly how a real parent zone hands off to a child zone.

If no delegation applies, the server looks up the exact queried name in its zone data. A few different cases get handled:

If the name doesn't exist at all, the server responds with NXDOMAIN and includes the zone's SOA record in the authority section so the client knows where the negative answer came from.

If the query is for NS records and the name has them, the server answers directly and throws in any glue A records it knows about for the nameserver hostnames.

If the query is for some type other than CNAME but the name has CNAME records, the server follows the chain. It includes every CNAME hop in the answer section as it goes, and if the chain eventually resolves to a name that has the requested record type within the same zone, that gets included too. A visited-set tracks names already seen so a circular CNAME chain can't cause an infinite loop.

For MX queries, the server includes A records for the mail exchange hostname in the additional section when it has them, which saves the client an extra lookup.

For everything else, the server just returns whatever records match the queried type. If the name exists but has nothing matching the requested type, it returns an empty answer with the SOA in the authority section.

All responses from the authoritative path set the AA (Authoritative Answer) flag in the header.

## Recursive Resolver

For names outside the local zone, `resolve` handles everything. It starts at the root server IP that was passed in on the command line and works its way down the DNS tree one delegation at a time, querying nameservers until one of them actually answers the question.

The loop is capped at 30 iterations so a pathological delegation chain can't spin forever. Recursion depth is also capped at 20, which matters because resolving the IP address of a nameserver can itself require a recursive resolution — the cap prevents stack overflow if something circular or absurdly deep shows up.

At each step, the resolver looks at what came back from the current server. If there's an actual answer in the response (records in the answer section that match what was asked for), resolution is done and the answer gets returned. If the response only has NS records in the authority section (a referral), the resolver picks the most specific one — the NS records whose owner name is the longest match for the queried name — and figures out where to send the next query.

Getting IPs for the next nameservers happens in a priority order: first it checks the glue records in the additional section of the referral (fastest, no extra lookup needed), then the cache (still no network needed), and finally a full recursive resolution of the nameserver's A record (slowest, but necessary when there's no glue and nothing cached). This fallback chain means the resolver can bootstrap itself even when glue is missing.

CNAME handling in recursive mode works differently than in authoritative mode. If a query comes back with only CNAME records in the answer section and nothing else, the resolver extracts the final target of the chain and recursively resolves that target for the original query type, then appends the results to the response. This produces a complete answer for the client rather than forcing them to issue a second query.

NS queries get special treatment. Sometimes the authoritative server for a zone puts the NS records in the authority section rather than the answer section. The resolver detects this case — NS records in the authority section whose owner name matches exactly what was asked for — and promotes them into the answer section before returning.

## Caching

Every resource record that comes back from an upstream server during resolution gets stored in the cache if its TTL is greater than zero. The cache key is the combination of the lowercased name and the record type. Each entry stores the original record alongside its absolute expiration time (current wall clock time plus TTL at the time it was received). When a cached entry is retrieved, the remaining TTL is computed on the fly and clamped to a minimum of 1 so clients always see a positive TTL even for records about to expire.

Records with TTL 0 are never cached at all. This is intentional and important — some servers return zero-TTL records specifically to prevent caching, and respecting that is both correct behavior and necessary for the truncation test to work properly (the truncation test server returns TTL 0 records).

Cache lookups happen at the very top of `resolve`, before any network traffic goes out. If the answer is already cached, the resolver builds a response from the cached records and returns immediately. This is what makes repeated queries for the same name fast and what makes TTL countdown work correctly across multiple clients asking for the same thing.

The cache is a plain Python dictionary accessed under a threading lock. On every access, expired entries are pruned from the relevant bucket so the cache doesn't grow unboundedly with stale data.

## Bailiwick Enforcement

Bailiwick checking is the main defense against cache poisoning. The idea is simple: a nameserver is only allowed to tell you about names it's actually responsible for. If you're talking to the nameserver for `foo.com`, it has no business sending you records for `bar.com` or `google.com`. A malicious or misconfigured server that tries to inject records for names outside its zone needs to be ignored.

After every response comes back from an upstream server, `apply_bailiwick` filters all three sections — answer, authority, and additional — and drops any record whose owner name isn't within the current zone being queried. The bailiwick starts as `.` (the root, which accepts everything) and narrows to the current zone as referrals are followed. So once you're talking to the nameserver for `foo.com`, only records under `foo.com` are accepted from it.

This filter runs before anything gets cached, so poisoned records never make it into the cache in the first place.

## Handling Truncated Responses and Network Failures

`send_query` is the function that actually puts a UDP packet on the wire and waits for an answer. It will retry up to three times if anything goes wrong. The things that count as "something going wrong" are: the socket times out waiting for a response, the response can't be parsed as a valid DNS message, any other exception during send or receive, or the response arrives with the TC (truncated) flag set.

TC=1 means the server had more data to send than fit in a single UDP datagram and cut off the response. The standard way to handle this is TCP fallback, but in this project retrying the UDP query is sufficient because the test infrastructure's truncation plugin truncates every other response — so the retry will land on the non-truncated turn. Discarding the truncated response and retrying gets a complete answer without needing TCP at all.

All the retry logic is silent. There's no partial data returned, no error logged to the client. If all retries fail, `send_query` returns `None`, and the caller treats that as a dead server and either moves on to the next candidate or returns SERVFAIL.

## Error Handling at Every Layer

At the outermost level, `handle_request` wraps the entire request processing pipeline in a try/except. Any exception that somehow makes it through the inner handlers causes a SERVFAIL response to be sent back to the client. The server never goes silent on a client — even if something completely unexpected happens internally, the client gets a response.

The very first thing `handle_request` does is try to parse the incoming datagram. If it can't be parsed as a valid DNS message, the function returns immediately with no response. There's no point sending an error back to something that isn't speaking DNS.

Requests that have anything other than exactly one question are rejected with SERVFAIL. Requests that don't have the RD (Recursion Desired) flag set and aren't for the authoritative zone are rejected with SERVFAIL — the server only does recursion for clients that ask for it.

Inside `resolve`, the depth cap and iteration cap are the guards against infinite loops. If resolution hits either limit, the function returns `None` and the client gets SERVFAIL. This is correct behavior: if the DNS tree is so deep or so circular that an answer can't be found within reasonable bounds, admitting failure is better than hanging or returning a wrong answer.

## Challenges and Debugging

Getting the first few tests passing was straightforward — parsing the zone file, answering simple A queries, returning NXDOMAIN. That part mostly worked on the first try because the logic is simple and the test cases are easy to reason about. The difficulty came in layers as the tests got more complex.

The first real headache was sub-zone delegation in the authoritative handler. The initial version just looked up the queried name directly in the zone dictionary and returned whatever it found. That worked fine for names that actually lived in the zone, but it completely ignored the fact that some ancestor of the name might have NS records, meaning authority had already been handed off to a different server. The fix — walking up the label hierarchy and checking for NS records at each level before doing anything else — seemed obvious in hindsight, but getting the loop bounds right took a few tries. Off-by-one errors in how many labels to check caused it to either miss delegations or falsely delegate queries for the zone apex itself.

CNAME chaining caused problems in two separate places. In the authoritative handler, the first version would follow CNAMEs but forget to check whether the chain had looped back on itself. A zone file with a circular CNAME — `a CNAME b`, `b CNAME a` — would spin forever. Adding the visited set fixed that. In the recursive resolver the problem was different: when an upstream server returned a CNAME in the answer section, the code initially just returned that CNAME to the client and called it done. That's technically correct DNS behavior, but the test harness expected the resolver to chase the chain and return the final answer. Recognizing "all CNAMEs, no final answer" as the signal to recurse on the target took some reading of the test output to figure out.

The NS query handling in recursive mode was probably the most confusing bug to track down. Sometimes the authoritative server for a zone puts NS records in the authority section instead of the answer section, even when NS is exactly what was asked for. This happens when the server considers itself authoritative for the parent zone and treats the NS records as zone-cut information rather than data. The first version of the resolver would see no records in the answer section, find NS records in the authority section, and then try to follow them as a referral — looping forever or failing. The fix was to detect the case where the NS owner name in the authority section exactly matches the queried name and treat those records as the answer rather than a referral.

Bailiwick enforcement broke things in a way that was hard to diagnose at first. When it was first added, it was too aggressive — it was filtering out records that were legitimately in-scope. The bug was in how the bailiwick string was initialized and updated as referrals were followed. The root bailiwick needs to be `.` to accept anything, and it needs to narrow to the zone name only after a referral is accepted. Getting those updates in the right order relative to when filtering happened took several runs against the bailiwick test to get right.

Getting IPs for nameservers during recursive resolution seemed simple at first — just use the glue records in the additional section. But glue isn't always present, and when it isn't, the resolver has to go find the nameserver's A record itself. The three-tier fallback (glue first, then cache, then full recursive resolve) worked in theory but caused problems when the recursive resolve for the nameserver's A record itself triggered another recursive resolve, which could hit the same issue. The depth limit exists partly to keep this from spiraling. There were also subtle cases where glue records arrived but weren't in-bailiwick and got filtered, leaving the resolver with no IPs and no way forward — which correctly falls through to the cache and recursive-resolve tiers.

Caching had two separate issues. The first was TTL decay: when a record is cached and then returned later, the TTL the client sees should reflect how much time has passed, not the original TTL. That part was straightforward. The harder issue was zero-TTL records. The truncation test server returns records with TTL 0 specifically so they won't be cached, and early versions of the cache code were storing them anyway because the check was `if rr.ttl < 0` instead of `if rr.ttl <= 0`. That caused the truncation test to behave incorrectly — the first response would be cached and subsequent queries would get a stale answer instead of going back to the server.

The truncation handling itself was simpler than expected. Since the test infrastructure truncates every other response from the same server, just retrying the same server on TC=1 gets a complete answer. The trickier part was making sure the retry didn't accidentally use a stale socket or mix up response data between retries. Using a fresh socket on each attempt inside the retry loop was the clean solution.

Thread safety turned out to be less of a problem than anticipated — mostly because Python's GIL provides some implicit protection — but the cache dictionary still needed an explicit lock because the read-modify-write pattern of pruning expired entries and updating the bucket is not atomic. The lock is a plain `threading.Lock` and is held for the minimum time necessary.

The concurrency test (test 14) — four clients firing forty requests each across multiple domains simultaneously — was the final stress test. The answers were all correct, but diagnosing why required looking at the full sequence of what each client sent and received and cross-referencing the timing. The test passes consistently once the caching and bailiwick logic was solid, but it was a useful forcing function for finding any remaining race conditions.
