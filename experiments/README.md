
# MADDNESS


## Building on our work and possible research projects

If you're interested in building on our work, the subsequent sections should help you with the details. If you think this kind of work is cool but aren't sure what concretely to pursue, here are some ideas I think are really promising:

- **Generalizing our approach to convolutions**. There's a lot of opportunity to reuse encodings here, which could allow the use of a more expensive encoding function (e.g., Bolt encoding, possibly augmented with OPQ-style rotations, learned permutations, or some block-diagonal rotations with intelligent block size).
- **Exploiting prior knowledge about one of the weight matrices (e.g., neural network weights)**. Instead of optimizing the centroids, you should just optimize the lookup tables directly. This is guaranteed to do no worse on the training set and is still such a simple model that it probably won't overfit. Basically a free win in exchange for stronger assumptions. You could probably change like 30 lines of code, rerun our experiments + recreate our figures, and have a paper.
- **Applying this approach to deep learning inference**. This will take a lot of CUDA kernel writing, but should be pretty straightforward. The unknowns here basically match those of more traditional quantization---i.e., how much you can get away with in each layer, how much fine-tuning to do, etc. You might even be able to get a training-time speedup if you swap in the ops early, although you'll hit some engineering difficulties here (e.g., torch DDP can't deal with param tensor types or shapes changing).
- **Throwing more compute at learning the encoding function**. We have a simple greedy algorithm for constructing the tree that runs in seconds or minutes on a laptop. But people are often willing to wait hours or days. You could probably exploit this extra time to explore more trees, more split dimensions, etc. Especially if you use a slightly more expensive encoding function.
- **Better theoretical guarantees**. Our current generalization guarantee gets *looser* as the training matrix has more low-dimensional structure. It should actually be the opposite, at least for a fixed Frobenius norm. The challenge is showing that the tree learning procedure manages to exploit this structure, rather than just assuming worst-case behavior.
- **More general theoretical guarantees**. This one is the most speculative. People talk about the constant of *exact* matrix multiplication. But what about approximate matrix multiplication? What's the limit here, as a function of amount of approximation error, characteristics of the matrices being multiplied, etc?


## Python

There's a Python implementation of the algorithm [here](https://github.com/dblalock/bolt/blob/45454e6cfbc9300a43da6770abf9715674b47a0f/experiments/python/vq_amm.py#L273), but no Python wrappers, sadly. Contributions welcome. Since SWIG is [constantly breaking](https://github.com/dblalock/bolt/issues/4), I'd suggest just creating PyTorch custom ops.

Unless you need timing measurements, I would definitely recommend using this Python implementation for reproducing our accuracy numbers or building on / comparing to our work.

The file that runs everything is [amm_main.py](https://github.com/dblalock/bolt/blob/master/experiments/python/amm_main.py). It's not pretty, but it is easy to hook in new approximate matrix multiplication algorithms and configure hparam values to sweep over. Run it with `python -m python.amm_main` from the `experiments` directory. See [amm.py](https://github.com/dblalock/bolt/blob/e4fe7b6dc08e9370644c3227f8199ca3c271af13/experiments/python/amm.py#L47) for many example method implementations.

The file with the top-level MADDNESS implementation is [vq_amm.py](https://github.com/dblalock/bolt/blob/45454e6cfbc9300a43da6770abf9715674b47a0f/experiments/python/vq_amm.py#L273), though the guts are mostly in [vquantizers.py](https://github.com/dblalock/bolt/blob/master/experiments/python/vquantizers.py) and [clusterize.py](https://github.com/dblalock/bolt/blob/master/experiments/python/clusterize.py).


### Datasets

- [Caltech-101](http://www.vision.caltech.edu/Image_Datasets/Caltech101/) - The dataset we extracted image patches from.
- [UCR Time Series Archive](https://www.cs.ucr.edu/~eamonn/time_series_data/) - The corpus we used to compare AMM methods across many datasets in a unified manner.
- [CIFAR-10 and CIFAR-100](https://www.cs.toronto.edu/~kriz/cifar.html).

You'll need to modify `experiments/python/matmul_datasets.py` to point to the paths where you store them all.

## C++

To run any of the MADDNESS C++ code, use [cpp/test/main.cpp](https://github.com/dblalock/bolt/blob/master/cpp/test/main.cpp). I used Catch tests for everything because it gives you a nice CLI and makes debugging easy.

You can run it with no arguments, but you probably want to pass some tags for which tests to run--the full suite takes a while. You probably want something like `[amm] ~[old]` to just run the MADDNESS tests and exclude legacy / debugging tests.

You'll need to use the same steps as for the Bolt C++ code, described below, to get it all to build.


### Running experiments and making plots

So the bad news is that the data collection pipeline is kind of an abomination. To get timing numbers, I ran the appropriate C++ tests in Catch, which dump results to stdout. I then pasted these into various results files (see `experiments/results/amm`). The file `experiments/python/amm_results2.py` then parses these and joins them with the accuracy numbers from `experiments/python/amm_main.py` to get dataframes of time, acc, std deviation, etc. The file `experiments/python/amm_figs2.py` pulls these results to make figures. If you just want to rerun the existing results and not add your own, you should be able to just use `amm_figs2.py` and not touch `amm_results2.py`. NOTE: `amm_results2.py` uses joblib to cache the results for each method, since they take a while to process; make sure you wipe the cache whenever you want to read in newer numbers.

It of course would have been way nicer to have Pybind11 wrappers for everything instead of catch > stdout > parsing, but...deadlines.


# Bolt


This page describes how to reproduce the experimental results reported in Bolt's KDD paper.

## Install Dependencies

To run the experiments, you will first need to obtain the following tools / libraries and datasets.

### C++ Code

- [Bazel](http://bazel.build), Google's open-source build system

Alternative: Xcode. This is the way I developed everything, and should "just work" if you open `cpp/xcode/Bolt/Bolt/Bolt.xcodeproj`.

### Python Code:
- [Joblib](https://github.com/joblib/joblib) - for caching function output
- [Scikit-learn](https://github.com/scikit-learn/scikit-learn) - for k-means
- [Pandas](http://pandas.pydata.org) - for storing results and reading in data
- [Seaborn](https://github.com/mwaskom/seaborn) - for plotting, if you want to reproduce our figures

### Datasets:
- [Sift1M](http://corpus-texmex.irisa.fr) - 1 million SIFT descriptors. This page also hosts the Sift10M, Sift100M, Sift1B, and Gist1M datasets (not used in the paper)
- [Convnet1M](https://github.com/una-dinosauria/stacked-quantizers) - 1 million convnet image descriptors
- [MNIST](http://yann.lecun.com/exdb/mnist/) - 60,000 images of handwritten digits; perhaps the most widely used dataset in machine learning
- [LabelMe](http://www.cs.toronto.edu/~norouzi/research/mlh/) - 22,000 GIST descriptors of images

Additional datasets not used in the paper:
- [Glove](https://github.com/erikbern/ann-benchmarks) - 1 million word embeddings generated using [Glove](https://nlp.stanford.edu/projects/glove/)
- [Gist](http://corpus-texmex.irisa.fr) - 1 million GIST descriptors. Available from the same source as Sift1M.
- [Deep1M](http://sites.skoltech.ru/compvision/projects/aqtq/) - PCA projections of deep descriptors for 1 million images

Note that, in order to avoid the temptation to cherry pick, we did not run experiments on these until after deciding to use only the first four datasets in the paper.

## Reproduce Timing / Throughput results

The timing scripts available include:
 - `time_bolt.sh`: Computes Bolt's time/throughput when encoding data vectors and queries, scanning through a dataset given an already-encoded query, and answering queries (encode query + scan).
 - `time_pq_opq.sh`: This is the same as `time_bolt.sh`, but the results are for Product Quantization (PQ) and Optimized Product Quantization (OPQ). There is no OPQ scan because the algorithm is identical to PQ once the dataset and query are both encoded.
 - `time_popcount.sh`: Computes the time/throughput for Hamming distance computation using the `popcount` instruction.
 - `time_matmul_square.sh`: Computes the time/throughput for matrix multiplies using square matrices.
 - `time_matmul_tall.sh`: Computes the time/throughput for matrix multiplies using tall, skinny matrices.

To reproduce the timing experiments using these scripts:

1. Run the appropriate shell script in this directory. E.g.:
    ```
    $ bash time_bolt.sh
    ```

2. Since we want to be certain that the compiler unrolls all the relevant loops / inlines code equally for all algorithms, the number of bytes used in encodings and lengths of vectors are constants at the tops of the corresponding files. To assess all the combinations of encoding / vector lengths used in the paper, you will have to modify these constants and rerun the tests multiple times. Ideally, this would be automated or handled via template parameters, but it isn't; pull requests welcome.

| Test          |   File        | Constants and Experimental Values
|:----------    |:----------    |:----------
| time_bolt.sh  | cpp/test/quantize/profile_bolt.cpp | M = {8,16,32}, ncols={64, 128, 256, 512, 1024}
| time_pq_opq.sh  | cpp/test/quantize/profile_pq.cpp | M = {8,16,32}, ncols={64, 128, 256, 512, 1024}
| time_popcount.sh  | cpp/test/quantize/profile_multicodebook.cpp | M = {8,16,32}
| time_matmul_square.sh  | cpp/test/quantize/profile_bolt.cpp | sizes = [64, 128, 256, 512, 1024, 2048, 4096, 8192]
| time_matmul_tall.sh  | cpp/test/quantize/profile_bolt.cpp | sizes = [1, 16, 32, 64, 128, 256, 512, 1024, 2048]

The `sizes` for the `time_matmul_*` scripts only need to be modified once, since the code iterates over all provided sizes; the tests are just set to run a subset of the sizes at the moment so that one can validate that it runs without having to wait for minutes.


## Reproduce Accuracy Results

### Clean Datasets

Unlike the timing experiments, the accuracy experiments depend on the aforementioned datasets. We do not own these datasets, so instead of distributing them ourselves, we provide code to convert them to the necessary formats and precompute the true nearest neighbors.

After downloading any dataset from the above links, start an IPython notebook:

```bash
    cd python/
    ipython notebook &
```

then open `munge_data.ipynb` and execute the code under the dataset's heading. This code will save a query matrix, a database matrix, and a ground truth nearest neighbor indices matrix. For some datasets, it will also save a training database matrix. Computing the true nearest neighbors will take a while for datasets wherein these neighbors are not already provided.

Once you have the above matrices for the datasets saved, go to python/datasets.py and fill in the appropriate path constants so that they can be loaded on your machine.

### Run the Accuracy Experiments

To compute the correlations between the approximate and true dot products:
```bash
$ bash correlations_dotprod.sh
```

To compute the correlations between the approximate and true squared Euclidean distances:
```bash
$ bash correlations_l2.sh
```

To compute the recall@R:
```bash
$ bash recall_at_r.sh
```

## Compare to Our Results

All raw results are in the `results/` directory. `summary.csv` files store aggregate metrics, and `all_results.csv` files store metrics for individual queries. The latter are mostly present so that we can easily plot confidence intervals with Seaborn.


## Notes

At present, Bolt has only been tested with Clang on OS X. The timing / throughput results in the paper are based on compiling with Xcode, not Bazel, although this shouldn't make a difference.

Feel free to contact us with any and all questions.

<!-- Also, because this code is a cleaned-up version of what we originally ran, there is a small but nonzero chance we've subtly broken something or diminished the performance. Please don't hesitate to contact us if you believe this is the case. -->
