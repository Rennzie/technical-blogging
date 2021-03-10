# Performance and the why of COGs

The story for COGs at Hummingbird is really about cost benefit. In our case, cloud compute and storage cost versus higher resolution images in our web platform. On platform performance is key as well.

Diving into storage and processing performance we noticed map tiles become more expensive than COGs as soon as you tile over zl 20. As we've mentioned we need resolution down to zl 24 or 25 for our 5cm GSD products. Some back of the envelope calculations show we'd be saving up to 45% in storage costs if we used COGs. There's another win there too, when its all said and done, you only need one remaining file - the COG. When it comes to processing, COGs are often up to 10x faster to generate than tiles. In our books this is a massive win.

Measuring on platform performance is a bit trickier and subjective. What is a considered a "fast" load to a user navigating a lettuce field. Is that the most important metric or is seeming greater detail at lower zoom level important. We are actively evaluating this question, but for the time being we are happy with the perceived performance of COGs when compare to map tiles. If we get a more concrete answer we'll be sure to write up a new post on our learnings.
