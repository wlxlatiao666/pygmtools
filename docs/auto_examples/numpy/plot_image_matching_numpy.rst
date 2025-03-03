
.. DO NOT EDIT.
.. THIS FILE WAS AUTOMATICALLY GENERATED BY SPHINX-GALLERY.
.. TO MAKE CHANGES, EDIT THE SOURCE PYTHON FILE:
.. "auto_examples/numpy/plot_image_matching_numpy.py"
.. LINE NUMBERS ARE GIVEN BELOW.

.. only:: html

    .. note::
        :class: sphx-glr-download-link-note

        Click :ref:`here <sphx_glr_download_auto_examples_numpy_plot_image_matching_numpy.py>`
        to download the full example code

.. rst-class:: sphx-glr-example-title

.. _sphx_glr_auto_examples_numpy_plot_image_matching_numpy.py:


========================================
Matching Image Keypoints by QAP Solvers
========================================

This example shows how to match image keypoints by graph matching solvers provided by ``pygmtools``.
These solvers follow the Quadratic Assignment Problem formulation and can generally work out-of-box.
The matched images can be further processed for other downstream tasks.

.. GENERATED FROM PYTHON SOURCE LINES 11-17

.. code-block:: default


    # Author: Runzhong Wang <runzhong.wang@sjtu.edu.cn>
    #         Wenzheng Pan <pwz1121@sjtu.edu.cn>
    #
    # License: Mulan PSL v2 License








.. GENERATED FROM PYTHON SOURCE LINES 19-30

.. note::
    The following solvers support QAP formulation, and are included in this example:

    * :func:`~pygmtools.classic_solvers.rrwm` (classic solver)

    * :func:`~pygmtools.classic_solvers.ipfp` (classic solver)

    * :func:`~pygmtools.classic_solvers.sm` (classic solver)

    * :func:`~pygmtools.neural_solvers.ngm` (neural network solver)


.. GENERATED FROM PYTHON SOURCE LINES 30-42

.. code-block:: default

    import numpy as np # numpy backend
    import cv2 as cv
    import pygmtools as pygm
    import matplotlib.pyplot as plt # for plotting
    from matplotlib.patches import ConnectionPatch # for plotting matching result
    import scipy.io as sio # for loading .mat file
    import scipy.spatial as spa # for Delaunay triangulation
    from sklearn.decomposition import PCA as PCAdimReduc
    import itertools
    from PIL import Image
    pygm.BACKEND = 'numpy' # set numpy as backend for pygmtools








.. GENERATED FROM PYTHON SOURCE LINES 43-50

Load the images
----------------
Images are from the Willow Object Class dataset (this dataset also available with the Benchmark of ``pygmtools``,
see :class:`~pygmtools.dataset.WillowObject`).

The images are resized to 256x256.


.. GENERATED FROM PYTHON SOURCE LINES 50-62

.. code-block:: default

    obj_resize = (256, 256)
    img1 = Image.open('../data/willow_duck_0001.png')
    img2 = Image.open('../data/willow_duck_0002.png')
    kpts1 = np.array(sio.loadmat('../data/willow_duck_0001.mat')['pts_coord'])
    kpts2 = np.array(sio.loadmat('../data/willow_duck_0002.mat')['pts_coord'])
    kpts1[0] = kpts1[0] * obj_resize[0] / img1.size[0]
    kpts1[1] = kpts1[1] * obj_resize[1] / img1.size[1]
    kpts2[0] = kpts2[0] * obj_resize[0] / img2.size[0]
    kpts2[1] = kpts2[1] * obj_resize[1] / img2.size[1]
    img1 = img1.resize(obj_resize, resample=Image.BILINEAR)
    img2 = img2.resize(obj_resize, resample=Image.BILINEAR)








.. GENERATED FROM PYTHON SOURCE LINES 63-65

Visualize the images and keypoints


.. GENERATED FROM PYTHON SOURCE LINES 65-80

.. code-block:: default

    def plot_image_with_graph(img, kpt, A=None):
        plt.imshow(img)
        plt.scatter(kpt[0], kpt[1], c='w', edgecolors='k')
        if A is not None:
            for x, y in zip(np.nonzero(A)[0], np.nonzero(A)[1]):
                plt.plot((kpt[0, x], kpt[0, y]), (kpt[1, x], kpt[1, y]), 'k-')

    plt.figure(figsize=(8, 4))
    plt.subplot(1, 2, 1)
    plt.title('Image 1')
    plot_image_with_graph(img1, kpts1)
    plt.subplot(1, 2, 2)
    plt.title('Image 2')
    plot_image_with_graph(img2, kpts2)




.. image-sg:: /auto_examples/numpy/images/sphx_glr_plot_image_matching_numpy_001.png
   :alt: Image 1, Image 2
   :srcset: /auto_examples/numpy/images/sphx_glr_plot_image_matching_numpy_001.png
   :class: sphx-glr-single-img





.. GENERATED FROM PYTHON SOURCE LINES 81-86

Build the graphs
-----------------
Graph structures are built based on the geometric structure of the keypoint set. In this example,
we refer to `Delaunay triangulation <https://en.wikipedia.org/wiki/Delaunay_triangulation>`_.


.. GENERATED FROM PYTHON SOURCE LINES 86-97

.. code-block:: default

    def delaunay_triangulation(kpt):
        d = spa.Delaunay(kpt.T)
        A = np.zeros((len(kpt[0]), len(kpt[0])))
        for simplex in d.simplices:
            for pair in itertools.permutations(simplex, 2):
                A[pair] = 1
        return A

    A1 = delaunay_triangulation(kpts1)
    A2 = delaunay_triangulation(kpts2)








.. GENERATED FROM PYTHON SOURCE LINES 98-100

We encode the length of edges as edge features


.. GENERATED FROM PYTHON SOURCE LINES 100-105

.. code-block:: default

    A1 = ((np.expand_dims(kpts1, 1) - np.expand_dims(kpts1, 2)) ** 2).sum(axis=0) * A1
    A1 = (A1 / A1.max()).astype(np.float32)
    A2 = ((np.expand_dims(kpts2, 1) - np.expand_dims(kpts2, 2)) ** 2).sum(axis=0) * A2
    A2 = (A2 / A2.max()).astype(np.float32)








.. GENERATED FROM PYTHON SOURCE LINES 106-108

Visualize the graphs


.. GENERATED FROM PYTHON SOURCE LINES 108-116

.. code-block:: default

    plt.figure(figsize=(8, 4))
    plt.subplot(1, 2, 1)
    plt.title('Image 1 with Graphs')
    plot_image_with_graph(img1, kpts1, A1)
    plt.subplot(1, 2, 2)
    plt.title('Image 2 with Graphs')
    plot_image_with_graph(img2, kpts2, A2)




.. image-sg:: /auto_examples/numpy/images/sphx_glr_plot_image_matching_numpy_002.png
   :alt: Image 1 with Graphs, Image 2 with Graphs
   :srcset: /auto_examples/numpy/images/sphx_glr_plot_image_matching_numpy_002.png
   :class: sphx-glr-single-img





.. GENERATED FROM PYTHON SOURCE LINES 117-121

Extract node features
----------------------
Let's adopt the SIFT method to extract node features.


.. GENERATED FROM PYTHON SOURCE LINES 121-137

.. code-block:: default

    np_img1 = np.array(img1, dtype=np.float32)
    np_img2 = np.array(img2, dtype=np.float32)

    def detect_sift(img):
        sift = cv.SIFT_create() 
        gray = cv.cvtColor(img, cv.COLOR_BGR2GRAY)
        img8bit = cv.normalize(gray, None, 0, 255, cv.NORM_MINMAX).astype('uint8')
        kpt = sift.detect(img8bit, None) 
        kpt, feat = sift.compute(img8bit, kpt) 
        return kpt, feat

    sift_kpts1, feat1 = detect_sift(np_img1)
    sift_kpts2, feat2 = detect_sift(np_img2)
    sift_kpts1 = np.round(cv.KeyPoint_convert(sift_kpts1).T).astype(int)
    sift_kpts2 = np.round(cv.KeyPoint_convert(sift_kpts2).T).astype(int)








.. GENERATED FROM PYTHON SOURCE LINES 138-140

Normalize the features


.. GENERATED FROM PYTHON SOURCE LINES 140-144

.. code-block:: default

    num_features = feat1.shape[1]
    feat1 = feat1 / np.expand_dims(np.linalg.norm(feat1, axis=1), 1).repeat(128, axis=1)
    feat2 = feat2 / np.expand_dims(np.linalg.norm(feat2, axis=1), 1).repeat(128, axis=1)








.. GENERATED FROM PYTHON SOURCE LINES 145-147

Extract node features by nearest interpolation


.. GENERATED FROM PYTHON SOURCE LINES 147-164

.. code-block:: default

    rounded_kpts1 = np.round(kpts1).astype(int)
    rounded_kpts2 = np.round(kpts2).astype(int)

    idx_1, idx_2 = [], []
    for i in range(rounded_kpts1.shape[1]):
        y1 = np.where(sift_kpts1[1] == sift_kpts1[1][np.abs(sift_kpts1[1] - rounded_kpts1[1][i]).argmin()])
        y2 = np.where(sift_kpts2[1] == sift_kpts2[1][np.abs(sift_kpts2[1] - rounded_kpts2[1][i]).argmin()])
        t1 = sift_kpts1[0][y1]
        t2 = sift_kpts2[0][y2]
        x1 = np.where(sift_kpts1[0] == t1[np.abs(t1 - rounded_kpts1[0][i]).argmin()])
        x2 = np.where(sift_kpts2[0] == t2[np.abs(t2 - rounded_kpts2[0][i]).argmin()])
        idx_1.append(np.intersect1d(x1, y1)[0])
        idx_2.append(np.intersect1d(x2, y2)[0])

    node1 = feat1[idx_1, :] # shape: NxC
    node2 = feat2[idx_2, :] # shape: NxC








.. GENERATED FROM PYTHON SOURCE LINES 165-176

Build affinity matrix
----------------------
We follow the formulation of Quadratic Assignment Problem (QAP):

.. math::

    &\max_{\mathbf{X}} \ \texttt{vec}(\mathbf{X})^\top \mathbf{K} \texttt{vec}(\mathbf{X})\\
    s.t. \quad &\mathbf{X} \in \{0, 1\}^{n_1\times n_2}, \ \mathbf{X}\mathbf{1} = \mathbf{1}, \ \mathbf{X}^\top\mathbf{1} \leq \mathbf{1}

where the first step is to build the affinity matrix (:math:`\mathbf{K}`)


.. GENERATED FROM PYTHON SOURCE LINES 176-182

.. code-block:: default

    conn1, edge1 = pygm.utils.dense_to_sparse(A1)
    conn2, edge2 = pygm.utils.dense_to_sparse(A2)
    import functools
    gaussian_aff = functools.partial(pygm.utils.gaussian_aff_fn, sigma=1) # set affinity function
    K = pygm.utils.build_aff_mat(node1, edge1, conn1, node2, edge2, conn2, edge_aff_fn=gaussian_aff)








.. GENERATED FROM PYTHON SOURCE LINES 183-189

Visualization of the affinity matrix. For graph matching problem with :math:`N` nodes, the affinity matrix
has :math:`N^2\times N^2` elements because there are :math:`N^2` edges in each graph.

.. note::
    The diagonal elements are node affinities, the off-diagonal elements are edge features.


.. GENERATED FROM PYTHON SOURCE LINES 189-193

.. code-block:: default

    plt.figure(figsize=(4, 4))
    plt.title(f'Affinity Matrix (size: {K.shape[0]}$\\times${K.shape[1]})')
    plt.imshow(K, cmap='Blues')




.. image-sg:: /auto_examples/numpy/images/sphx_glr_plot_image_matching_numpy_003.png
   :alt: Affinity Matrix (size: 100$\times$100)
   :srcset: /auto_examples/numpy/images/sphx_glr_plot_image_matching_numpy_003.png
   :class: sphx-glr-single-img


.. rst-class:: sphx-glr-script-out

 .. code-block:: none


    <matplotlib.image.AxesImage object at 0x7f8fbbfb17f0>



.. GENERATED FROM PYTHON SOURCE LINES 194-198

Solve graph matching problem by RRWM solver
-------------------------------------------
See :func:`~pygmtools.classic_solvers.rrwm` for the API reference.


.. GENERATED FROM PYTHON SOURCE LINES 198-200

.. code-block:: default

    X = pygm.rrwm(K, kpts1.shape[1], kpts2.shape[1])








.. GENERATED FROM PYTHON SOURCE LINES 201-203

The output of RRWM is a soft matching matrix. Hungarian algorithm is then adopted to reach a discrete matching matrix.


.. GENERATED FROM PYTHON SOURCE LINES 203-205

.. code-block:: default

    X = pygm.hungarian(X)








.. GENERATED FROM PYTHON SOURCE LINES 206-211

Plot the matching
------------------
The correct matchings are marked by green, and wrong matchings are marked by red. In this example, the nodes are
ordered by their ground truth classes (i.e. the ground truth matching matrix is a diagonal matrix).


.. GENERATED FROM PYTHON SOURCE LINES 211-223

.. code-block:: default

    plt.figure(figsize=(8, 4))
    plt.suptitle('Image Matching Result by RRWM')
    ax1 = plt.subplot(1, 2, 1)
    plot_image_with_graph(img1, kpts1, A1)
    ax2 = plt.subplot(1, 2, 2)
    plot_image_with_graph(img2, kpts2, A2)
    for i in range(X.shape[0]):
        j = np.argmax(X[i]).item()
        con = ConnectionPatch(xyA=kpts1[:, i], xyB=kpts2[:, j], coordsA="data", coordsB="data",
                              axesA=ax1, axesB=ax2, color="red" if i != j else "green")
        plt.gca().add_artist(con)




.. image-sg:: /auto_examples/numpy/images/sphx_glr_plot_image_matching_numpy_004.png
   :alt: Image Matching Result by RRWM
   :srcset: /auto_examples/numpy/images/sphx_glr_plot_image_matching_numpy_004.png
   :class: sphx-glr-single-img





.. GENERATED FROM PYTHON SOURCE LINES 224-232

Solve by other solvers
-----------------------
We could also do a quick benchmarking of other solvers on this specific problem.

IPFP solver
^^^^^^^^^^^
See :func:`~pygmtools.classic_solvers.ipfp` for the API reference.


.. GENERATED FROM PYTHON SOURCE LINES 232-246

.. code-block:: default

    X = pygm.ipfp(K, kpts1.shape[1], kpts2.shape[1])

    plt.figure(figsize=(8, 4))
    plt.suptitle('Image Matching Result by IPFP')
    ax1 = plt.subplot(1, 2, 1)
    plot_image_with_graph(img1, kpts1, A1)
    ax2 = plt.subplot(1, 2, 2)
    plot_image_with_graph(img2, kpts2, A2)
    for i in range(X.shape[0]):
        j = np.argmax(X[i]).item()
        con = ConnectionPatch(xyA=kpts1[:, i], xyB=kpts2[:, j], coordsA="data", coordsB="data",
                              axesA=ax1, axesB=ax2, color="red" if i != j else "green")
        plt.gca().add_artist(con)




.. image-sg:: /auto_examples/numpy/images/sphx_glr_plot_image_matching_numpy_005.png
   :alt: Image Matching Result by IPFP
   :srcset: /auto_examples/numpy/images/sphx_glr_plot_image_matching_numpy_005.png
   :class: sphx-glr-single-img


.. rst-class:: sphx-glr-script-out

 .. code-block:: none

    /Users/guoziao/Desktop/pygmtools-git/pygmtools/numpy_backend.py:304: RuntimeWarning: invalid value encountered in true_divide
      t0 = alpha / beta




.. GENERATED FROM PYTHON SOURCE LINES 247-251

SM solver
^^^^^^^^^^^
See :func:`~pygmtools.classic_solvers.sm` for the API reference.


.. GENERATED FROM PYTHON SOURCE LINES 251-266

.. code-block:: default

    X = pygm.sm(K, kpts1.shape[1], kpts2.shape[1])
    X = pygm.hungarian(X)

    plt.figure(figsize=(8, 4))
    plt.suptitle('Image Matching Result by SM')
    ax1 = plt.subplot(1, 2, 1)
    plot_image_with_graph(img1, kpts1, A1)
    ax2 = plt.subplot(1, 2, 2)
    plot_image_with_graph(img2, kpts2, A2)
    for i in range(X.shape[0]):
        j = np.argmax(X[i]).item()
        con = ConnectionPatch(xyA=kpts1[:, i], xyB=kpts2[:, j], coordsA="data", coordsB="data",
                              axesA=ax1, axesB=ax2, color="red" if i != j else "green")
        plt.gca().add_artist(con)




.. image-sg:: /auto_examples/numpy/images/sphx_glr_plot_image_matching_numpy_006.png
   :alt: Image Matching Result by SM
   :srcset: /auto_examples/numpy/images/sphx_glr_plot_image_matching_numpy_006.png
   :class: sphx-glr-single-img





.. GENERATED FROM PYTHON SOURCE LINES 267-278

NGM solver
^^^^^^^^^^^
See :func:`~pygmtools.neural_solvers.ngm` for the API reference.

.. note::
    The NGM solvers are pretrained on a different problem setting, so their performance may seem inferior.
    To improve their performance, you may change the way of building affinity matrices, or try finetuning
    NGM on the new problem.

The NGM solver pretrained on Willow dataset:


.. GENERATED FROM PYTHON SOURCE LINES 278-293

.. code-block:: default

    X = pygm.ngm(K, kpts1.shape[1], kpts2.shape[1], pretrain='willow')
    X = pygm.hungarian(X)

    plt.figure(figsize=(8, 4))
    plt.suptitle('Image Matching Result by NGM (willow pretrain)')
    ax1 = plt.subplot(1, 2, 1)
    plot_image_with_graph(img1, kpts1, A1)
    ax2 = plt.subplot(1, 2, 2)
    plot_image_with_graph(img2, kpts2, A2)
    for i in range(X.shape[0]):
        j = np.argmax(X[i]).item()
        con = ConnectionPatch(xyA=kpts1[:, i], xyB=kpts2[:, j], coordsA="data", coordsB="data",
                              axesA=ax1, axesB=ax2, color="red" if i != j else "green")
        plt.gca().add_artist(con)




.. image-sg:: /auto_examples/numpy/images/sphx_glr_plot_image_matching_numpy_007.png
   :alt: Image Matching Result by NGM (willow pretrain)
   :srcset: /auto_examples/numpy/images/sphx_glr_plot_image_matching_numpy_007.png
   :class: sphx-glr-single-img


.. rst-class:: sphx-glr-script-out

 .. code-block:: none


    Downloading to /Users/guoziao/Library/Caches/pygmtools/ngm_willow_numpy.npy...

    Downloading to /Users/guoziao/Library/Caches/pygmtools/ngm_willow_numpy.npy...
      0%|          | 0/14319 [00:00<?, ?it/s]    100%|##########| 14.0k/14.0k [00:00<00:00, 2.66MB/s]




.. GENERATED FROM PYTHON SOURCE LINES 294-296

The NGM solver pretrained on VOC dataset:


.. GENERATED FROM PYTHON SOURCE LINES 296-310

.. code-block:: default

    X = pygm.ngm(K, kpts1.shape[1], kpts2.shape[1], pretrain='voc')
    X = pygm.hungarian(X)

    plt.figure(figsize=(8, 4))
    plt.suptitle('Image Matching Result by NGM (voc pretrain)')
    ax1 = plt.subplot(1, 2, 1)
    plot_image_with_graph(img1, kpts1, A1)
    ax2 = plt.subplot(1, 2, 2)
    plot_image_with_graph(img2, kpts2, A2)
    for i in range(X.shape[0]):
        j = np.argmax(X[i]).item()
        con = ConnectionPatch(xyA=kpts1[:, i], xyB=kpts2[:, j], coordsA="data", coordsB="data",
                              axesA=ax1, axesB=ax2, color="red" if i != j else "green")
        plt.gca().add_artist(con)



.. image-sg:: /auto_examples/numpy/images/sphx_glr_plot_image_matching_numpy_008.png
   :alt: Image Matching Result by NGM (voc pretrain)
   :srcset: /auto_examples/numpy/images/sphx_glr_plot_image_matching_numpy_008.png
   :class: sphx-glr-single-img






.. rst-class:: sphx-glr-timing

   **Total running time of the script:** ( 0 minutes  9.602 seconds)


.. _sphx_glr_download_auto_examples_numpy_plot_image_matching_numpy.py:

.. only:: html

  .. container:: sphx-glr-footer sphx-glr-footer-example


    .. container:: sphx-glr-download sphx-glr-download-python

      :download:`Download Python source code: plot_image_matching_numpy.py <plot_image_matching_numpy.py>`

    .. container:: sphx-glr-download sphx-glr-download-jupyter

      :download:`Download Jupyter notebook: plot_image_matching_numpy.ipynb <plot_image_matching_numpy.ipynb>`


.. only:: html

 .. rst-class:: sphx-glr-signature

    `Gallery generated by Sphinx-Gallery <https://sphinx-gallery.github.io>`_
