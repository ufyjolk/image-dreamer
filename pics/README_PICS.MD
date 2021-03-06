##Information about pictures, copyright & file names
The original pictures here are semi-randomly selected pictures from the Internet to which I do not have any copyrights and have been used and published here for purely non-commercial and educational purposes. If you are the owner of any of those pictures and would like it to be deleted, please let me know by using github and the files you will claim will be deleted asap.

Some of the photos were converted from PNG or BMP file format to JPG, but no record of this changes was made. Most of the pictures were resized to a semi-random resolution to help the script run faster, check the impact of the resolution on the script's speed and avoid potential memory problems and no record of these changes was made. All modifications to these files was made using IrfanView.


The file names are coded in the following way:
-dream.png => standard settings [1], standard set[0], 1st try
-dream2.png => standard settings [1], standard set, 2nd try
-dream3.png => 1st mod of settings [2], standard set, 1st try
-dream4.png => 2nd mod of settings [3] /1st layer test/, standard set, 1st try
-dream5.png => 3nd mod of settings [4] /2nd layer test/, standard set, 1st try
-dream6.png => 4nd mod of settings [5] /3rd layer test/, standard set, 1st try
-dream7.png => 5nd mod of settings [6] /4th layer test/, standard set, 1st try

If the file is in the JPG file format and has no "-dream-" addition to its name, it mean that this is the original file

NOTE: Currently you can find here original pictures and those created in the 1st and 2nd try (-dream.png and -dream2.png). This repo will be updated.

##Notes
[0] By "standard set" I mean that the script has not been trained with any additional picture sets, than originally implemented by its creator (see the original github repo of which this one is a fork).


[1] Default dreamify.py

# imports and basic notebook setup
from cStringIO import StringIO
import numpy as np
import scipy.ndimage as nd
import PIL.Image
from IPython.display import clear_output, Image, display
from google.protobuf import text_format

import caffe

import sys

def savearray(a, filename, fmt='png'):
    a = np.uint8(np.clip(a, 0, 255))
    with open(filename, 'wb') as f:
        PIL.Image.fromarray(a).save(f, fmt)
        #display(Image(data=f.getvalue()))

model_path = '/home/vagrant/caffe/models/bvlc_googlenet/' # substitute your path here
net_fn   = model_path + 'deploy.prototxt'
param_fn = model_path + 'bvlc_googlenet.caffemodel'

# Patching model to be able to compute gradients.
# Note that you can also manually add "force_backward: true" line to "deploy.prototxt".
model = caffe.io.caffe_pb2.NetParameter()
text_format.Merge(open(net_fn).read(), model)
model.force_backward = True
open('tmp.prototxt', 'w').write(str(model))

net = caffe.Classifier('tmp.prototxt', param_fn,
                       mean = np.float32([104.0, 116.0, 122.0]), # ImageNet mean, training set dependent
                       channel_swap = (2,1,0)) # the reference model has channels in BGR order instead of RGB

# a couple of utility functions for converting to and from Caffe's input image layout
def preprocess(net, img):
    return np.float32(np.rollaxis(img, 2)[::-1]) - net.transformer.mean['data']
def deprocess(net, img):
    return np.dstack((img + net.transformer.mean['data'])[::-1])

def make_step(net, step_size=1.5, end='inception_4c/output', jitter=32, clip=True):
    '''Basic gradient ascent step.'''

    src = net.blobs['data'] # input image is stored in Net's 'data' blob
    dst = net.blobs[end]

    ox, oy = np.random.randint(-jitter, jitter+1, 2)
    src.data[0] = np.roll(np.roll(src.data[0], ox, -1), oy, -2) # apply jitter shift

    net.forward(end=end)
    dst.diff[:] = dst.data  # specify the optimization objective
    net.backward(start=end)
    g = src.diff[0]
    # apply normalized ascent step to the input image
    src.data[:] += step_size/np.abs(g).mean() * g

    src.data[0] = np.roll(np.roll(src.data[0], -ox, -1), -oy, -2) # unshift image

    if clip:
        bias = net.transformer.mean['data']
        src.data[:] = np.clip(src.data, -bias, 255-bias)

def deepdream(net, base_img, iter_n=10, octave_n=4, octave_scale=1.4, end='inception_4c/output', clip=True, **step_params):
    # prepare base images for all octaves
    octaves = [preprocess(net, base_img)]
    for i in xrange(octave_n-1):
        octaves.append(nd.zoom(octaves[-1], (1, 1.0/octave_scale,1.0/octave_scale), order=1))

    src = net.blobs['data']
    detail = np.zeros_like(octaves[-1]) # allocate image for network-produced details
    for octave, octave_base in enumerate(octaves[::-1]):
        h, w = octave_base.shape[-2:]
        if octave > 0:
            # upscale details from the previous octave
            h1, w1 = detail.shape[-2:]
            detail = nd.zoom(detail, (1, 1.0*h/h1,1.0*w/w1), order=1)

        src.reshape(1,3,h,w) # resize the network's input image size
        src.data[0] = octave_base+detail
        for i in xrange(iter_n):
            make_step(net, end=end, clip=clip, **step_params)

            # visualization
            vis = deprocess(net, src.data[0])
            if not clip: # adjust image contrast if clipping is disabled
                vis = vis*(255.0/np.percentile(vis, 99.98))
            #showarray(vis)
            print octave, i, end, vis.shape
            clear_output(wait=True)

        # extract details produced on the current octave
        detail = src.data[0]-octave_base
    # returning the resulting image
    return deprocess(net, src.data[0])

if 'keys'==sys.argv[1]:
    print "\n".join( net.blobs.keys())
    sys.exit(0)

img = np.float32(PIL.Image.open(sys.argv[1]))
end_name = sys.argv[3] if len(sys.argv)>3 else 'inception_4c/output'
iter_n = int(sys.argv[4]) if len(sys.argv)>4 else 10
octaves = int(sys.argv[5]) if len(sys.argv)>5 else 4
oct_scale = float(sys.argv[6]) if len(sys.argv)>6 else 1.4
_=deepdream(net, img, end=end_name, iter_n=iter_n, octave_n=octaves, octave_scale=oct_scale)
savearray(_, sys.argv[2])


[2] 1st mod dreamify.py, line 62 change:
def deepdream(net, base_img, iter_n=15, octave_n=8, octave_scale=1.4, end='inception_4c/output', clip=True, **step_params): #"inter_n=10, octave_n=8" was in the original


[3]
def deepdream(net, base_img, iter_n=15, octave_n=8, octave_scale=1.4, end='inception_3a/1x1', clip=True, **step_params): #"inter_n=10, octave_n=4, octave_scale=1.4, end='inception_4c/output" was in the original


[4]
def deepdream(net, base_img, iter_n=15, octave_n=8, octave_scale=1.4, end='inception_3a/3x3', clip=True, **step_params): #"end='inception_3a/1x1" was in the last version


[5]
def deepdream(net, base_img, iter_n=15, octave_n=8, octave_scale=1.4, end='inception_3a/3x3_reduce', clip=True, **step_params): #"end='inception_3a/3x3" was in the last version


[6]
def deepdream(net, base_img, iter_n=15, octave_n=8, octave_scale=1.4, end='inception_3a/5x5', clip=True, **step_params): #"end='inception_3a/3x3_reduce" was in the last version
