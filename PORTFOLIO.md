# VibeFinder

VibeFinder is an explainable Python music recommender for CodePath AI110. It ranks a 20-track classroom catalog against listener preferences and explains the score for each track. The Project 4 extension checks genre coverage, filters weak matches, and audits the final recommendations.

[Source code and reproducible demo](https://github.com/Somama12/vibefinder-applied-ai-system)

This project shows how I approach reliability in an AI system. I wanted the recommender to explain its choices and tell the listener when the catalog could support only part of a request. Testing full, partial, and unsupported requests made the tradeoff visible: a stricter match rule gives a clearer result, but it can also leave out tracks a listener might enjoy. My next step would be to use feedback from real listeners to test the weights and threshold.
