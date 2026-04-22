## High-Level Approach

So when a client sends it a query, it figures out which mode applies and handles it accordingly. If the name being asked about lives inside the loaded zone file then the server answers directly from its own records. It will follow the typcial recursive resolver process if it doesnt know the answer by going from name server to name server until it recieves an answer.

Everything runs on UDP and on threads so that things can run in paralllel. The cache is shared by each thread, but I made sure that the cache updates atomically and by one thread at a time so it stays consistent.

like you said we should follow the "assume that the remote server is a psycopath trying to kill you" I made sure to react to issues as if I was getting bad data or was getting tricked. The server never trusts data from the outside world. Every response from an upstream nameserver gets filtered before anything in it is believed or stored. Also we handle errors gracefully by doing a SERVFAIL so the program never crashes and deals with issues in a controlled way.

## Authoritative Mode

When a query comes in for a name that falls inside the server's own zone, "handle authorization" will takes over. The first thing it does is check whether the name actually belongs to a sub-zone that has been delegated out by walking up the labels of the queried name one step at a time and checking whether any ancestor has NS records in the zone data. If it finds one then it will return a referral that puts the NS records in the authority section and if the server knows the IP addresses of those nameservers, then it will include them in the additional section

If no delegation applies then the server looks up the exact queried name in its zone data

1. If the name doesn't exist at all then the server will respond with NXDOMAIN and include the zone's SOA record in the authority section so the client can keep track of where the negative answer came from.

2. To deal with cyclic dependencies that may come up, if the name server ever has a glue records associated with a name server it will automatically include them

3. If the query is for some type other than CNAME but the name has CNAME records, the server follows the chain. I also make sure to keep track of all visited hops and CNAME records that pop up just in case there are some self referential aliasing of sorts happening and an infinite loop doesn't happen.

4. For MX queries the server includes A records for the mail exchange hostname in the additional section when it has them.
5. For everything else, the server just returns whatever records match the queried type. If the name exists but has nothing matching the requested type, it returns an empty answer with the SOA in the authority section.

## Recursive Resolver

this functions exactly like the nameserver described in class does

for testing I originally had to cap it becuase I was getting issues with infinite depth recursive resolution. By the nature of a recursive resolver if there is case where glue records failed or CNAME references itself, then testing would take forever. It stays in the code because it is a good indicator that something went wrong and tests pass with it still in.

I also made to make sure that I used the cache and glue records when possible to prevent unecesarry lookups when it could be avoidable. We only do recursive resolution when this happens.

CNAMES are handled as expected with us looking for aliases and checking if there is any instance where we should query for the alias. We also append the CNAME to our oriiginal query to prevent a second lookup.

Sometimes the authoritative server will put NS records in the authoritative section instead of the answer section. we can check for this happening just by seeing if the domain name matches the NS response. if this happens we just move the NS authoritative answers into the answer section

## Caching

Every resource record that comes back from an upstream server during resolution gets stored in the cache if its TTL is greater than zero. When a cached entry is retrieved, the remaining TTL is computed on the fly and clamped to a minimum of 1 so clients always see a positive TTL even for records about to expire.

For the truncation test, I found that requests will specifically specify a TTL of 0 which means not to cache. If these responses were cached anyways this test was failing.

I used python dictionaries which has a built in threading lock which I utlizied to ensure that stale data is never accessed or stored.

## Bailiwick Enforcement

After every response comes back from an upstream server "apply_bailiwick" filters all three sections answer, authority, and additional and drops any record whose owner name isn't within the current zone being queried. The bailiwick starts as . and narrows to the current zone as referrals are followed. So once you're talking to the nameserver for "foo.com", only records under "foo.com" are accepted from it.

## Handling Truncated Responses and Network Failures

"send_query" is the function that actually puts a UDP packet on the wire and waits for an answer. if the socket times out waiting for a response, the response can't be parsed as a valid DNS message, any other exception during send or receive, or the response arrives with TC set, then it will retry up to 3 times

if the retries all fail, either move on return SERVFAIL, which is another component of trying to do graceful failure.

## Error Handling at Every Layer

At the outermost level "handle_request" wraps the entire request processing pipeline in a try except. Any exception that somehow makes it through the inner handlers causes a SERVFAIL response to be sent back to the client. The server never goes silent on a client — even if something completely unexpected happens internally, the client gets a response.

The very first thing "handle_request" does is try to parse the incoming datagram. if it can't parse then it simply ignores the request and moves on

Requests that have anything other than exactly one question are rejected with SERVFAIL. Requests that dont have RD set and aren't for the authoritative zone are rejected with SERVFAIL

Inside "resolve", the depth cap and iteration cap are the guards against infinite loops like i mentioned before. if we reach these limits then we return nothing and SERVFAIL is returned

## Challenges and Debugging

Getting the first few tests passing was fine because the logic is simple and the test cases are easy to reason about

The first real headache was sub-zone delegation in the authoritative handler. The initial version just looked up the queried name directly in the zone dictionary and returned whatever it found which worked fine for names that actually lived in the zone, but it completely ignored the fact that some ancestor of the name might have NS records. we fixed this by checking for NS records at each level in the heirarchy before doing anything else The fix — walking up the label hierarchy. I kept missing records due to not checking completely (due to off by one errors and such) but eventually I was able to reach every part of the tree and search fully which helped me pass this test case

CNAME chaining caused problems in two separate places. In the authoritative handler, the first version would follow CNAMEs but forget to check whether the chain had looped back on itself. liek I mentioned before we keep track of CNAMEs we have seen "our visited set" which fixed that. for the recursive resolver when an upstream server returned a CNAME in the answer section, the code initially just returned that CNAME to the client and called it done. I think this is fine technically but the test cases wanted us to chase down the chain and return the final answer at the end of the chain. In the test outputs the thing that indicated that was the message "all CNAMEs, no final answer"

The NS query handling in recursive mode was probably the most confusing bug to track dow. sometimes the authoritative server for a zone puts NS records in the authority section instead of the answer section even when NS is exactly what was asked for. This happens when the server considers itself authoritative for the parent zone and treats the NS records as zones instead of data. By adding logic to detect when the NS owner's name in the authority section exactly matches the queried name and treat those records as the answer rather than a referral

Bailiwick enforcement broke things in a way that was hard to diagnose at first. When it was first added, it was too aggressive — it was filtering out too many records. The bug was in how the bailiwick string was initialized and updated as referrals were followed. The root bailiwick needs to be . to accept anything, and it needs to narrow to the zone name only after a referral is accepted. Getting those updates in the right order relative to when filtering happened took several runs against the bailiwick test to get right.

Getting IPs for nameservers during recursive resolution seemed simple at first however glue records weren't always present requiring the resolver to fetch them itself. the depth limit exists partly to keep this from spiraling. There were also subtle cases where glue records arrived but weren't in-bailiwick and got filtered, leaving the resolver with no IPs and no way forward which correctly falls through to the cache and recursive resolve tiers.

I had two issues with caching which inluded ttl decay when a record was cached and then returned later and the TTL the client sees should reflect how much time had passed not the original TTL. there were also zero TTL records like before this was a simple fix becuase I wasn't expecting TTLs of 0. and my check would check TTL > 0 rather than TTL >= 0 which led to unexpected issues.
