### Problem:

Given a list of posts, compute the top 5 related posts for each post based on the number of shared tags.

<details>
<summary> Steps </summary>

-   Read the posts JSON file.
-   Iterate over the posts and populate a map containing: `tag -> List<int>`, with the int representing the post index of each post with that tag.
-   Iterate over the posts and for each post:
    -   Create a map: `PostIndex -> int` to track the number of shared tags
    -   For each tag, Iterate over the posts that have that tag
    -   For each post, increment the shared tag count in the map.
-   Sort the related posts by the number of shared tags.
-   Write the top 5 related posts for each post to a new JSON file.
</details>

### Run Benchmark

See [setup.md](setup.md) for complete guide.

```bash
./run.sh go | rust | python | all

# windows (powershell)
./run.ps1 go | rust | python | all
# OR
pwsh ./run.ps1 go | rust | python | all

# Docker (check the dockerfiles/base.Dockerfile for available variables)
./gen_dockerfile.sh -b go | rust | python | all
# THEN

./docker_run.sh go | rust | python | all
# OR use the image directly
docker run -e TEST_NAME=go -it --rm go_databench
```

<details>
<summary> Rules </summary>

<h3>No:</h3>

-   FFI (including assembly inlining)
-   Unsafe code blocks
-   Custom benchmarking
-   Disabling runtime checks (bounds etc)
-   Specific hardware targeting
-   SIMD for single threaded solutions
-   Hardcoding number of posts
-   Lazy evaluation (Unless results are computed at runtime and timed)
-   Computation Caching

<h3>Must:</h3>

-   Support up to 100,000 posts
-   Support UTF8 strings
-   Parse json at runtime
-   Support up to 100 tags
-   Represent tags as strings
-   Be production ready
-   Use less than 8GB of memory
</details>

### Single Core Results ([raw data](https://github.com/jinyus/related_post_gen/blob/main/results))

##### VM Specs ( OVHcloud bm-s1-8vCPU-32GB-RAM-Ubuntu 26.04 )

| Language         | Time (5k posts)                       | 20k posts                              | 60k posts                           | Total     |
| ---------------- | ------------------------------------- | -------------------------------------- | ----------------------------------- | --------- |
| Rust | $\textsf{\color{lightgreen}15.12 ms}$ | $\textsf{\color{lightgreen}113.96 ms}$ | $\textsf{\color{lightgreen}422.55 ms}$ | 551.64 ms |
| Go | 19.44 ms | 161.11 ms | 618.98 ms | 799.53 ms |
| R (tabulate) | 405.69 ms | 2.32 s | 8.22 s | 10.96 s |
| R | 2.10 s | 15.20 s | 57.80 s | 75.10 s |
| Common Lisp (SBCL) | OOM | OOM | OOM | N/A |

### Multicore Results

| Language       | Time (5k posts) | 20k posts        | 60k posts        | Total     |
| -------------- | --------------- | ---------------- | ---------------- | --------- | 
| Rust Concurrent | 13.79 ms | $\textsf{\color{lightgreen}82.44 ms}$ | $\textsf{\color{lightgreen}263.84 ms}$ | 360.08 ms |
| Go Concurrent | $\textsf{\color{lightgreen}10.99 ms}$ | 90.54 ms | 344.41 ms | 445.94 ms |

### Highly Optimised Results

| Language       | Time (5k posts) | 20k posts        | 60k posts        | Total     |
| -------------- | --------------- | ---------------- | ---------------- | --------- | 
| _Go HO_[^1] | $\textsf{\color{lightgreen}6.87 ms}$ | $\textsf{\color{lightgreen}19.93 ms}$ | $\textsf{\color{lightgreen}33.96 ms}$ | 60.75 ms |

> **Disclaimer:** These results are not directly comparable to the other implementations. The HO entries hardcode the top-5 output (`NUM_TOP_ITEMS`/`TopN`/`SVector{5}`) and exploit constraints of the benchmark dataset that don't generalize:
> - **Rust HO** uses 8-bit tag IDs, so it fails with more than 256 unique tags, and posts with more than 8 tags fall back to the standard, unoptimized algorithm.
> - **D HO** requires posts to have at most 8 tags (it aborts otherwise) and uses lookup tables limited to 256 entries.
> - **Julia HO** always returns exactly 5 related posts via a specialized data structure.
> - **Go HO** requires posts to have at most 8 tags and the dataset to have at most 256 unique tags (it exits with an error otherwise), and always returns exactly 5 related posts.
>
> They only produce these speeds for this specific dataset/shape of input; an arbitrary number of tags would break them or require changes.

<details>
<summary> Old Results with details (on my machine) </summary>

| Language   | Processing Time | Total (+ I/O) | Details                                                                                                                                                                                                                                                                                         |
| ---------- | --------------- | ------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Rust       | -               | 4.5s          | Initial                                                                                                                                                                                                                                                                                         |
| Rust v2    | -               | 2.60s         | Replace std HashMap with fxHashMap by [phazer99](https://www.reddit.com/r/rust/comments/16plgok/comment/k1rtr4x/?utm_source=share&utm_medium=web2x&context=3)                                                                                                                                   |
| Rust v3    | -               | 1.28s         | Preallocate and reuse map and unstable sort by [vdrmn](https://www.reddit.com/r/rust/comments/16plgok/comment/k1rzo7g/?utm_source=share&utm_medium=web2x&context=3) and [Darksonn](https://www.reddit.com/r/rust/comments/16plgok/comment/k1rzwdx/?utm_source=share&utm_medium=web2x&context=3) |
| Rust v4    | -               | 0.13s         | Use Post index as key instead of Pointer and Binary Heap by [RB5009](https://www.reddit.com/r/rust/comments/16plgok/comment/k1s5ea0/?utm_source=share&utm_medium=web2x&context=3)                                                                                                               |
| Rust v5    | 38ms            | 52ms          | Rm hashing from loop and use vec[count] instead of map[index]count by RB5009                                                                                                                                                                                                                    |
| Rust v6    | 23ms            | 36ms          | Optimized Binary Heap Ops by [scottlamb](https://github.com/jinyus/related_post_gen/pull/12)                                                                                                                                                                                                    |
| Rust Rayon | 9ms             | 22ms          | Parallelize by [masmullin2000](https://github.com/jinyus/related_post_gen/pull/4)                                                                                                                                                                                                               |
| Rust Rayon | 8ms             | 22ms          | Remove comparison out of hot loop                                                                                                                                                                                                                                                               |
| ⠀          | ⠀               | ⠀             | ⠀                                                                                                                                                                                                                                                                                               |
| Go         | -               | 1.5s          | Initial                                                                                                                                                                                                                                                                                         |
| Go v2      | -               | 80ms          | Add rust optimizations                                                                                                                                                                                                                                                                          |
| Go v3      | 56ms            | 70ms          | Use goccy/go-json                                                                                                                                                                                                                                                                               |
| Go v3      | 34ms            | 55ms          | Use generic binaryheap by [DrBlury](https://github.com/jinyus/related_post_gen/pull/7)                                                                                                                                                                                                          |
| Go v4      | 26ms            | 50ms          | Replace binary heap with custom priority queue                                                                                                                                                                                                                                                  |
| Go v5      | 20ms            | 43ms          | Remove comparison out of hot loop                                                                                                                                                                                                                                                               |
| Go Con     | 10ms            | 33ms          | Go concurrency by [tirprox](https://github.com/jinyus/related_post_gen/pull/17) and [DrBlury](https://github.com/jinyus/related_post_gen/pull/8)                                                                                                                                                |
| Go Con v2  | 5ms             | 29ms          | Use arena, use waitgroup, rm binheap by [DrBlury](https://github.com/jinyus/related_post_gen/pull/20)                                                                                                                                                                                           |
| ⠀          | ⠀               | ⠀             | ⠀                                                                                                                                                                                                                                                                                               |
| Python     | -               | 7.81s         | Initial                                                                                                                                                                                                                                                                                         |
| Python v2  | 1.35s           | 1.53s         | Add rust optimizations by [dave-andersen](https://github.com/jinyus/related_post_gen/pull/10)                                                                                                                                                                                                   |
| Numpy      | 0.57s           | 0.85s         | Numpy implementation by [Copper280z](https://github.com/jinyus/related_post_gen/pull/11)                                                                                                                                                                                                        |
| ⠀          | ⠀               | ⠀             | ⠀                                                                                                                                                                                                                                                                                               |
| Crystal    | 50ms            | 96ms          | Inital w/ previous optimizations                                                                                                                                                                                                                                                                |
| Crystal v2 | 33ms            | 72ms          | Replace binary heap with custom priority queue                                                                                                                                                                                                                                                  |
| ⠀          | ⠀               | ⠀             | ⠀                                                                                                                                                                                                                                                                                               |
| Odin       | 110ms           | 397ms         | Ported from golang code                                                                                                                                                                                                                                                                         |
| Odin v2    | 104ms           | 404ms         | Remove comparison out of hot loop                                                                                                                                                                                                                                                               |
| ⠀          | ⠀               | ⠀             | ⠀                                                                                                                                                                                                                                                                                               |
| Dart VM    | 125ms           | 530ms         | Ported from golang code                                                                                                                                                                                                                                                                         |
| Dart bin   | 274ms           | 360ms         | Compiled executable                                                                                                                                                                                                                                                                             |
| ⠀          | ⠀               | ⠀             | ⠀                                                                                                                                                                                                                                                                                               |
| Vlang      | 339ms           | 560ms         | Ported from golang code                                                                                                                                                                                                                                                                         |
| ⠀          | ⠀               | ⠀             | ⠀                                                                                                                                                                                                                                                                                               |
| Zig        | 80ms            | 110ms         | Provided by [akhildevelops](https://github.com/jinyus/related_post_gen/pull/30)                                                                                                                                                                                                                 |

</details>

[^1]: Uses specialized datastructures meant for demonstration purposes: [more](https://github.com/LilithHafner/Jokes/tree/main/SuperDataStructures.jl)

[^2]: Inko is currently in beta and optimizations haven't been applied. [more](https://github.com/jinyus/related_post_gen/pull/440#issuecomment-1816583612)