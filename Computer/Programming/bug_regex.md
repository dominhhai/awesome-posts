### Stack Exchange Network Status
Here we'll post updates on outages and maintenance windows for the Stack Exchange Network. You can also get status updates by following @StackStatus

### Outage Postmortem - July 20, 2016

#### Overview

On July 20, 2016 we experienced a 34 minute outage starting at 14:44 UTC. It took 10 minutes to identify the cause, 14 minutes to write the code to fix it, and 10 minutes to roll out the fix to a point where Stack Overflow became available again.

The direct cause was a malformed post that caused one of our regular expressions to consume high CPU on our web servers. The post was in the homepage list, and that caused the expensive regular expression to be called on each home page view. This caused the home page to stop responding fast enough. Since the home page is what our load balancer uses for the health check, the entire site became unavailable since the load balancer took the servers out of rotation.

#### Follow-up Actions

* Audit our regular expressions and post validation workflow for any similar issues
* Add controls to our load balancer to disable the healthcheck – as we believe everything but the home page would have been accessible if it wasn’t for the the health check
* Create a “what to do during an outage” checklist since our StackStatus Twitter notification was later than we would have liked (and a few other outage workflow items we would like to be more consistent on).

#### Technical Details

The regular expression was: `^[\s\u200c]+|[\s\u200c]+$` Which is intended to trim unicode space from start and end of a line. A simplified version of the Regex that exposes the same issue would be `\s+$` which to a human looks easy (“all the spaces at the end of the string”), but which means quite some work for a simple backtracking Regex engine. The malformed post contained roughly 20,000 consecutive characters of whitespace on a comment line that started with -- play happy sound for player to enjoy. For us, the sound was not happy.

If the string to be matched against contains 20,000 space characters in a row, but not at the end, then the Regex engine will start at the first space, check that it belongs to the `\s` character class, move to the second space, make the same check, etc. After the 20,000th space, there is a different character, but the Regex engine expected a space or the end of the string. Realizing it cannot match like this it backtracks, and tries matching `\s+$` starting from the second space, checking 19,999 characters. The match fails again, and it backtracks to start at the third space, etc.

So the Regex engine has to perform a “character belongs to a certain character class” check (plus some additional things) `20,000+19,999+19,998+…+3+2+1 = 199,990,000` times, and that takes a while. This is not classic catastrophic backtracking (talk on backtracking) (performance is `O(n²)`, not exponential, in length), but it was enough. This regular expression has been replaced with a substring function.

JULY 20, 2016 (8:47 PM)

> From [StackExchange](http://stackstatus.net/post/147710624694/outage-postmortem-july-20-2016)
