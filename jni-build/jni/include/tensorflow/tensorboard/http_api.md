# Tensorboard client-server HTTP API

## Runs, Tags, and Tag Types

TensorBoard data is organized around the concept of a `run`, which represents
all the related data thrown off by a single execution of TensorFlow, a `tag`,
which groups values of data that come from the same source within a TensorFlow
run, and `tag types`, which are our way of distinguishing different types of
data that have fundamentally different representations and should be processed
on different code paths. For example, a "train" run may have a `scalars`
tag that represents the learning rate, another `scalars` tag that
represents the value of the objective function, a `histograms` tag that reveals
information on weights in a particular layer over time, and an `images` tag that
shows input images flowing into the system. The "eval" run might have an
entirely different set of tag names, or some duplicated tag names.

The currently supported tag types are `scalars`, `images`, `histograms` and
`graph`. Each tag type corresponds to a route (documented below) for
retrieving tag data of that type.

All of the data provided comes from TensorFlow events files ('\*.tfevents\*'),
which are written using the SummaryWriter class
(tensorflow/python/training/summary_writer.py), and the data is generated by
summary ops (tensorflow/python/ops/summary_ops.py). The `scalars` come from
the `ScalarSummary` op, the `histograms` from the `HistogramSummary`, and the
`images` from `ImageSummary`. The tag type `graph` is special in that it is not
a collection of tags of that type, but a boolean denoting if there is a graph
definition associated with the run. The tag is provided to the summary
op (usually as a constant).

## `/runs`

Returns a dictionary mapping from `run name` (quoted string) to dictionaries
mapping from all available tagTypes to a list of tags of that type available for
the run. Think of this as a comprehensive index of all of the data available
from the TensorBoard server. Here is an example:

{
  "train_run": {
    "histograms": ["foo_histogram", "bar_histogram"],
    "compressedHistograms": ["foo_histogram", "bar_histogram"],
    "scalars": ["xent", "loss", "learning_rate"],
    "images": ["input"],
    "graph": true
  },
  "eval": {
    "histograms": ["foo_histogram", "bar_histogram"],
    "compressedHistograms": ["foo_histogram", "bar_histogram"],
    "scalars": ["precision", "recall"],
    "images": ["input"],
    "graph": false
  }
}

Note that the same tag may be present for many runs. It is not guaranteed that
they will have the same meaning across runs. It is also not guaranteed that they
will have the same tag type across different runs.

## '/scalars?run=foo&tag=bar'

Returns an array of event_accumulator.SimpleValueEvents ([wall_time, step,
value]) for the given run and tag. wall_time is seconds since epoch.

Example:
[
  [1443856985.705543, 1448, 0.7461960315704346],  # wall_time, step, value
  [1443857105.704628, 3438, 0.5427092909812927],
  [1443857225.705133, 5417, 0.5457325577735901],
  ...
]

If the format parameter is set to 'csv', the response will instead be in CSV
format:

    Wall time,step,value
    1443856985.705543,1448,0.7461960315704346
    1443857105.704628,3438,0.5427092909812927
    1443857225.705133,5417,0.5457325577735901


## '/histograms?run=foo&tag=bar'

Returns an array of event_accumulator.HistogramEvents ([wall_time, step,
HistogramValue]) for the given run and tag. A HistogramValue is [min, max, num,
sum, sum_squares, bucket_limit, bucket]. wall_time is seconds since epoch.

Annotated Example: (note - real data is higher precision)

[
  [
    1443871386.185149, # wall_time
    235166,            # step
    [
      -0.66,           # minimum value
      0.44,            # maximum value
      8.0,             # number of items in the histogram
      -0.80,           # sum of items in the histogram
      0.73,            # sum of squares of items in the histogram
      [-0.68, -0.62, -0.292, -0.26, -0.11, -0.10, -0.08, -0.07, -0.05,
       -0.0525, -0.0434, -0.039, -0.029, -0.026, 0.42, 0.47, 1.8e+308],
                       # the right edge of each bucket
     [0.0, 1.0, 0.0, 1.0, 0.0, 1.0, 0.0, 1.0, 0.0, 1.0, 0.0, 1.0, 0.0, 1.0, 0.0,
      1.0, 0.0]        # the number of elements within each bucket
     ]
   ]
 ]

## '/compressedHistograms?run=foo&tag=bar'

Returns an array of event_accumulator.CompressedHistogramEvents ([wall_time,
step, CompressedHistogramValues]) for the given run and tag.

CompressedHistogramValues is a list of namedtuples with each tuple specifying
a basis point (bps) as well as an interpolated value of the histogram value
at that basis point. A basis point is 1/100 of a percent.

The current compression strategy is to choose basis points that correspond to
the median and bands of 1SD, 2SD, and 3SDs around the median. Note that the
current compression strategy does not work well for representing multimodal
data -- this is something that will be improved in a later iteration.

Annotated Example: (note - real data is higher precision)

[
  [
    1441154832.580509,   # wall_time
    5,                   # step
    [  [0, -3.67],       # CompressedHistogramValue for 0th percentile
       [2500, -4.19],    # CompressedHistogramValue for 25th percentile
       [5000, 6.29],
       [7500, 1.64],
       [10000, 3.67]
    ]
  ],
  ...
]

## `/images?run=foo&tag=bar`

Gets a sample of ImageMetadatas for the given run and tag.

Returns an array of objects containing information about available images,
crucially including the query parameter that may be used to retrieve that image.
(See /individualImage for details.)

For example:
      {
        "width": 28,                 # width in pixels
        "height": 28,                # height in pixels
        "wall_time": 1440210599.246, # time in seconds since epoch
        "step": 63702821,            # number of steps that have passed
        "query": "index=0&tagname=input%2Fimage%2F2&run=train"
                                     # param for /individualImage
      }

## `/individualImage?{{query}}`

Retrieves an individual image. The image query should not be generated by the
frontend, but instead acquired from calling the /images route (the image
metadata objects contain the query to use). The response is the image itself
with mime-type 'image/png'.

Note that the query is not guaranteed to always refer to the same image even
within a single run, as images may be removed from the sampling reservoir and
replaced with other images. (See Notes for details on the reservoir sampling.)

An example call to this route would look like this:
/individualImage?index=0&tagname=input%2Fimage%2F2&run=train

## `/graph?run=foo`

Returns the graph definition for the given run in gzipped pbtxt format. The
graph is composed of a list of nodes, where each node is a specific TensorFlow
operation which takes as inputs other nodes (operations).

An example pbtxt response of graph with 3 nodes:
node {
  op: "Input"
  name: "A"
}
node {
  op: "Input"
  name: "B"
}
node {
  op: "MatMul"
  name: "C"
  input: "A"
  input: "B"
}

## Notes

All returned values, histograms, and images are returned in the order they were
written by Tensorflow (which should correspond to increasing `wall_time` order,
but may not necessarily correspond to increasing step count if the process had
to restart from a previous checkpoint).

The returned values may be downsampled using reservoir sampling, which is
configurable by the TensorBoard server. When downsampling occurs, the server
guarantees that different tags will all sample at the same sequence of indices,
so that if if there are two tags `A` and `B` which are related so that `A[i] ~
B[i]` for all `i`, then `D(A)[i] ~ D(B)[i]` for all `i`, where `D` represents
the downsampling operation.

The reservoir sampling puts an upper bound on the number of items that will be
returned for a given run-tag combination, and guarantees that all items are
equally likely to be in the final sample (ie it is a uniform distribution over
the values), with the proviso that the most recent individual item is always
included in the sample.

The reservoir sizes are configurable on a per-tag type basis.