# Results

Just like [last time with the router benchmark](https://github.com/l3tum/php-router-benchmark) I've chosen to write my own Container.
Unlike the Router though, we did use an actual Container for Victoria. That's not to say that it doesn't add overhead, but development
without one is just a PITA. Nevertheless I thought I could improve on current Containers.

For the interested: While building the benchmarks I came across an interesting [Optimization](./Curious_Optimization.md)
that I didn't think PHP would do. 

## Disclaimer

Usually when you have multiple implementations (Containers) of a Standard (PSR-11), these implementations will compete
on two metrics: Performance and Extra Features.

Extra Features are hard to quantify as the only two possible values are `I need it` and `I don't need it`.

That leaves us with Performance, which I tried to measure with these benchmarks here.

In order to compare Containers fairly a core set of features was devised that every Container had to implement
and was benchmarked on:
- Standard PSR-11 Features (Get/Has Service)
- Autowiring
- Aliasing
- Implementation-for-Interface

These benchmarks were performed with an auto-generated list of 100 Services.
I thought this was a good middle ground between small apps and big apps.

## Honorary Mentions

Three Container Implementations that were tested (and the benchmarks are in this repo) are Yii, League and Laminas.

League unfortunately does not offer a method to configure autowiring or service discovery ahead of time. This means
that League has to figure both out during the actual benchmark, resulting in abysmal performance. While this could be avoided by simply ``get``ing the Service
beforehand, this felt like a hack and not something that should be encouraged by a benchmark.

Although Yii does offer a method to configure autowiring and service discovery ahead of time,
it does not perform as well as one would expect. It actually performs around 10x worse than most other Containers.
In order to make the results accessible Yii was dropped from the comparisons.

Laminas does not offer autowiring out-of-the-box. While there are benchmarks with autowiring in this repo, they rely on a 
third-party library ([bluepsyduck/laminas-autowire-factory](https://github.com/BluePsyduck/laminas-autowire-factory)).
Since it does perform somewhat competitively though, I thought I'd leave it in.

## Actual Results Now

![Results](./images/ResultEarly.png)

Okay, you may wonder now: "Hey, Riaf isn't the fastest! What is this?!".
And I did so, too. How could anything be faster than my hobby project? That's illegal!

Well, truth be told, I knew of [Zen](https://github.com/woohoolabs/zen) beforehand. It was the inspiration for writing Riaf and for using `match`, as it is
using that itself as well. But how is it faster than Riaf?

It mostly comes down to the optimizations *around* the `match` itself. Riaf liberally used auto-generated factory methods
which did make the generated code easier to reason about, but obviously cost a ton of performance. Function calls should be avoided at all costs.
That's partly why PHP-DI performs worse: It's predominantly using function calls. 

With that in mind here's the updated results, after a [round of optimizations](https://github.com/l3tum/RiafCore/pull/31).

![Results](./images/ResultLater.png)

With these results in mind, it's quite clear that Riaf is, in addition to the fastest Router, also the fastest Container.
Interestingly though, unlike with Routers, Symfony performs surprisingly well and even beats out Zen sometimes.
That's most likely because Zen is using factory methods (like Riaf did before) and thus has this overhead.

Somewhat curiously PHP-DI also performs pretty well in comparison, which I did not expect at all.

Of course, as with the Router, increasing the number of Services will also rely more on the scaling of `match` and thus likely beat out other
Containers by a larger margin. If you want to read my (very naive) explanation on why `match` is faster you can look at the 
[router benchmark](https://github.com/l3tum/php-router-benchmark).

As you may have noticed, the absolute timings are much higher now. That's mostly because I reduced the times an individual benchmark
tries to fetch a Service. The reasoning behind that is pretty simple: Increasing the number of times a Service is fetched
flattens out the curve of timings and ultimately benchmarks a simple array-access, since the Services were defined as Singletons
and thus saved by the Container in an Array that it would use in later calls. I did not think of that behaviour before
running the first benchmarks, however re-running the benchmarks now it does not seem to change the overall scaling much.

## HAS Benchmarked

"Wait, why are we benchmarking `has` now?". I honestly thought it was wasted time to benchmark it, there can't be much variation, right?
And personally, I've never used the method anyways.

Surprisingly though, there are some differences between Containers.

![Results](./images/HASResult.png)

Unsurprisingly the two Containers making use of `match` (Zen and Riaf) top the benchmarks
with the others somewhat close together. The differences (and outliers) can be mostly explained by the way the Containers
save the different variants of keys (Aliases, Interface => Implementation, plain Service).

## Drawbacks

As with Routers, Riaf comes with a few drawbacks compared to Symfony specifically. Riaf does not support tagging (yet).
It also does not support calling methods on Instantiation (yet).

## Table of Results
                      
### GET

I've subtracted the results with 22??s for the GET graph so that the difference is clearer (as you can't set a baseline for the Y axis in PHPBench).

````
+--------------+--------------------------+-----------+----------+
| benchmark    | set                      | mem_peak  | mode     |
+--------------+--------------------------+-----------+----------+
| ZenBench     | GET Best Case            | 941.536kb | 27.018??s |
| ZenBench     | GET Worst Case Service99 | 941.536kb | 27.340??s |
| ZenBench     | GET Implementation       | 941.552kb | 35.695??s |
| ZenBench     | GET Interface            | 941.552kb | 35.845??s |
| ZenBench     | GET Alias                | 941.520kb | 28.436??s |
| ZenBench     | GET Service For Alias    | 941.536kb | 28.260??s |
| SymfonyBench | GET Best Case            | 941.544kb | 27.774??s |
| SymfonyBench | GET Worst Case Service99 | 941.544kb | 27.875??s |
| SymfonyBench | GET Implementation       | 941.560kb | 35.006??s |
| SymfonyBench | GET Interface            | 941.560kb | 35.144??s |
| SymfonyBench | GET Alias                | 941.528kb | 27.205??s |
| SymfonyBench | GET Service For Alias    | 941.544kb | 27.257??s |
| LaminasBench | GET Best Case            | 941.544kb | 30.048??s |
| LaminasBench | GET Worst Case Service99 | 941.544kb | 29.890??s |
| LaminasBench | GET Implementation       | 941.560kb | 38.957??s |
| LaminasBench | GET Interface            | 941.560kb | 38.558??s |
| LaminasBench | GET Alias                | 941.528kb | 30.849??s |
| LaminasBench | GET Service For Alias    | 941.544kb | 29.678??s |
| PhpDiBench   | GET Best Case            | 941.544kb | 26.438??s |
| PhpDiBench   | GET Worst Case Service99 | 941.544kb | 26.665??s |
| PhpDiBench   | GET Implementation       | 941.560kb | 34.614??s |
| PhpDiBench   | GET Interface            | 941.560kb | 34.908??s |
| PhpDiBench   | GET Alias                | 941.528kb | 27.000??s |
| PhpDiBench   | GET Service For Alias    | 941.544kb | 26.734??s |
| RiafBench    | GET Best Case            | 954.632kb | 26.205??s |
| RiafBench    | GET Worst Case Service99 | 954.632kb | 25.666??s |
| RiafBench    | GET Implementation       | 954.656kb | 34.667??s |
| RiafBench    | GET Interface            | 954.656kb | 34.283??s |
| RiafBench    | GET Alias                | 954.600kb | 25.579??s |
| RiafBench    | GET Service For Alias    | 954.632kb | 25.666??s |
+--------------+--------------------------+-----------+----------+
````

### HAS

````
+--------------+--------------------------+-----------+----------+
| benchmark    | set                      | mem_peak  | mode     |
+--------------+--------------------------+-----------+----------+
| ZenBench     | HAS Best Case            | 941.536kb | 0.039??s  |
| ZenBench     | HAS Worst Case Service99 | 941.536kb | 0.039??s  |
| ZenBench     | HAS Implementation       | 941.552kb | 0.040??s  |
| ZenBench     | HAS Interface            | 941.552kb | 0.039??s  |
| ZenBench     | HAS Alias                | 941.520kb | 0.038??s  |
| ZenBench     | HAS Service For Alias    | 941.536kb | 0.039??s  |
| SymfonyBench | HAS Best Case            | 941.544kb | 0.066??s  |
| SymfonyBench | HAS Worst Case Service99 | 941.544kb | 0.066??s  |
| SymfonyBench | HAS Implementation       | 941.560kb | 0.066??s  |
| SymfonyBench | HAS Interface            | 941.560kb | 0.082??s  |
| SymfonyBench | HAS Alias                | 941.528kb | 0.076??s  |
| SymfonyBench | HAS Service For Alias    | 941.544kb | 0.065??s  |
| LaminasBench | HAS Best Case            | 941.544kb | 0.058??s  |
| LaminasBench | HAS Worst Case Service99 | 941.544kb | 0.058??s  |
| LaminasBench | HAS Implementation       | 941.560kb | 0.059??s  |
| LaminasBench | HAS Interface            | 941.560kb | 0.095??s  |
| LaminasBench | HAS Alias                | 941.528kb | 0.096??s  |
| LaminasBench | HAS Service For Alias    | 941.544kb | 0.057??s  |
| PhpDiBench   | HAS Best Case            | 941.544kb | 0.052??s  |
| PhpDiBench   | HAS Worst Case Service99 | 941.544kb | 0.052??s  |
| PhpDiBench   | HAS Implementation       | 941.560kb | 0.052??s  |
| PhpDiBench   | HAS Interface            | 941.560kb | 0.053??s  |
| PhpDiBench   | HAS Alias                | 941.528kb | 0.051??s  |
| PhpDiBench   | HAS Service For Alias    | 941.544kb | 0.053??s  |
| RiafBench    | HAS Best Case            | 954.632kb | 0.038??s  |
| RiafBench    | HAS Worst Case Service99 | 954.632kb | 0.038??s  |
| RiafBench    | HAS Implementation       | 954.656kb | 0.038??s  |
| RiafBench    | HAS Interface            | 954.656kb | 0.037??s  |
| RiafBench    | HAS Alias                | 954.600kb | 0.037??s  |
| RiafBench    | HAS Service For Alias    | 954.632kb | 0.039??s  |
+--------------+--------------------------+-----------+----------+
````
