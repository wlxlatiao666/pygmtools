
.. DO NOT EDIT.
.. THIS FILE WAS AUTOMATICALLY GENERATED BY SPHINX-GALLERY.
.. TO MAKE CHANGES, EDIT THE SOURCE PYTHON FILE:
.. "auto_examples/paddle/plot_image_matching_paddle.py"
.. LINE NUMBERS ARE GIVEN BELOW.

.. only:: html

    .. note::
        :class: sphx-glr-download-link-note

        Click :ref:`here <sphx_glr_download_auto_examples_paddle_plot_image_matching_paddle.py>`
        to download the full example code

.. rst-class:: sphx-glr-example-title

.. _sphx_glr_auto_examples_paddle_plot_image_matching_paddle.py:


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


.. GENERATED FROM PYTHON SOURCE LINES 30-45

.. code-block:: default

    import paddle # pypaddle backend
    from paddle.vision.models import vgg16
    import pygmtools as pygm
    import matplotlib.pyplot as plt # for plotting
    from matplotlib.patches import ConnectionPatch # for plotting matching result
    import scipy.io as sio # for loading .mat file
    import scipy.spatial as spa # for Delaunay triangulation
    from sklearn.decomposition import PCA as PCAdimReduc
    import itertools
    import numpy as np
    from PIL import Image
    import warnings
    warnings.filterwarnings("ignore")
    pygm.BACKEND = 'paddle' # set default backend for pygmtools








.. GENERATED FROM PYTHON SOURCE LINES 46-53

Load the images
----------------
Images are from the Willow Object Class dataset (this dataset also available with the Benchmark of ``pygmtools``,
see :class:`~pygmtools.dataset.WillowObject`).

The images are resized to 256x256.


.. GENERATED FROM PYTHON SOURCE LINES 53-65

.. code-block:: default

    obj_resize = (256, 256)
    img1 = Image.open('../data/willow_duck_0001.png')
    img2 = Image.open('../data/willow_duck_0002.png')
    kpts1 = paddle.to_tensor(sio.loadmat('../data/willow_duck_0001.mat')['pts_coord'])
    kpts2 = paddle.to_tensor(sio.loadmat('../data/willow_duck_0002.mat')['pts_coord'])
    kpts1[0] = kpts1[0] * obj_resize[0] / img1.size[0]
    kpts1[1] = kpts1[1] * obj_resize[1] / img1.size[1]
    kpts2[0] = kpts2[0] * obj_resize[0] / img2.size[0]
    kpts2[1] = kpts2[1] * obj_resize[1] / img2.size[1]
    img1 = img1.resize(obj_resize, resample=Image.BILINEAR)
    img2 = img2.resize(obj_resize, resample=Image.BILINEAR)








.. GENERATED FROM PYTHON SOURCE LINES 66-68

Visualize the images and keypoints


.. GENERATED FROM PYTHON SOURCE LINES 68-83

.. code-block:: default

    def plot_image_with_graph(img, kpt, A=None):
        plt.imshow(img)
        plt.scatter(kpt[0], kpt[1], c='w', edgecolors='k')
        if A is not None:
            for idx in paddle.nonzero(A, as_tuple=False):
                plt.plot((kpt[0, idx[0]], kpt[0, idx[1]]), (kpt[1, idx[0]], kpt[1, idx[1]]), 'k-')

    plt.figure(figsize=(8, 4))
    plt.subplot(1, 2, 1)
    plt.title('Image 1')
    plot_image_with_graph(img1, kpts1)
    plt.subplot(1, 2, 2)
    plt.title('Image 2')
    plot_image_with_graph(img2, kpts2)




.. image-sg:: /auto_examples/paddle/images/sphx_glr_plot_image_matching_paddle_001.png
   :alt: Image 1, Image 2
   :srcset: /auto_examples/paddle/images/sphx_glr_plot_image_matching_paddle_001.png
   :class: sphx-glr-single-img





.. GENERATED FROM PYTHON SOURCE LINES 84-89

Build the graphs
-----------------
Graph structures are built based on the geometric structure of the keypoint set. In this example,
we refer to `Delaunay triangulation <https://en.wikipedia.org/wiki/Delaunay_triangulation>`_.


.. GENERATED FROM PYTHON SOURCE LINES 89-100

.. code-block:: default

    def delaunay_triangulation(kpt):
        d = spa.Delaunay(kpt.numpy().transpose())
        A = paddle.zeros((len(kpt[0]), len(kpt[0])))
        for simplex in d.simplices:
            for pair in itertools.permutations(simplex, 2):
                A[pair] = 1
        return A

    A1 = delaunay_triangulation(kpts1)
    A2 = delaunay_triangulation(kpts2)








.. GENERATED FROM PYTHON SOURCE LINES 101-103

We encode the length of edges as edge features


.. GENERATED FROM PYTHON SOURCE LINES 103-108

.. code-block:: default

    A1 = ((kpts1.unsqueeze(1) - kpts1.unsqueeze(2)) ** 2).sum(axis=0) * A1
    A1 = (A1 / A1.max()).cast(dtype=paddle.float32)
    A2 = ((kpts2.unsqueeze(1) - kpts2.unsqueeze(2)) ** 2).sum(axis=0) * A2
    A2 = (A2 / A2.max()).cast(dtype=paddle.float32)








.. GENERATED FROM PYTHON SOURCE LINES 109-111

Visualize the graphs


.. GENERATED FROM PYTHON SOURCE LINES 111-119

.. code-block:: default

    plt.figure(figsize=(8, 4))
    plt.subplot(1, 2, 1)
    plt.title('Image 1 with Graphs')
    plot_image_with_graph(img1, kpts1, A1)
    plt.subplot(1, 2, 2)
    plt.title('Image 2 with Graphs')
    plot_image_with_graph(img2, kpts2, A2)




.. image-sg:: /auto_examples/paddle/images/sphx_glr_plot_image_matching_paddle_002.png
   :alt: Image 1 with Graphs, Image 2 with Graphs
   :srcset: /auto_examples/paddle/images/sphx_glr_plot_image_matching_paddle_002.png
   :class: sphx-glr-single-img





.. GENERATED FROM PYTHON SOURCE LINES 120-124

Extract node features
----------------------
Let's adopt the VGG16 CNN model to extract node features.


.. GENERATED FROM PYTHON SOURCE LINES 124-135

.. code-block:: default

    vgg16_cnn = vgg16(pretrained=False, batch_norm=True) # no official pretrained paddle weight for vgg16_bn provided yet
    path = pygm.utils.download(filename='vgg16_bn.pdparams', \
                               url='https://drive.google.com/u/0/uc?export=download&confirm=Z-AR&id=11AGmtBrIZJLXJMk4Um9xQPai2EH7KjRY', \
                               md5='cf6079f3c8d16f42a93fc8f8b62e20d1') 
    vgg16_cnn.set_dict(paddle.load(path))
    paddle_img1 = paddle.to_tensor(np.array(img1, dtype=np.float32) / 256).transpose((2, 0, 1)).unsqueeze(0) # shape: BxCxHxW
    paddle_img2 = paddle.to_tensor(np.array(img2, dtype=np.float32) / 256).transpose((2, 0, 1)).unsqueeze(0) # shape: BxCxHxW
    with paddle.set_grad_enabled(False):
        feat1 = vgg16_cnn.features(paddle_img1)
        feat2 = vgg16_cnn.features(paddle_img2)








.. GENERATED FROM PYTHON SOURCE LINES 136-138

Normalize the features


.. GENERATED FROM PYTHON SOURCE LINES 138-146

.. code-block:: default

    num_features = feat1.shape[1]
    def l2norm(node_feat):
        return paddle.nn.functional.local_response_norm(
            node_feat, node_feat.shape[1] * 2, alpha=node_feat.shape[1] * 2, beta=0.5, k=0)

    feat1 = l2norm(feat1)
    feat2 = l2norm(feat2)








.. GENERATED FROM PYTHON SOURCE LINES 147-149

Up-sample the features to the original image size


.. GENERATED FROM PYTHON SOURCE LINES 149-152

.. code-block:: default

    feat1_upsample = paddle.nn.functional.interpolate(feat1, (obj_resize[1], obj_resize[0]), mode='bilinear')
    feat2_upsample = paddle.nn.functional.interpolate(feat2, (obj_resize[1], obj_resize[0]), mode='bilinear')








.. GENERATED FROM PYTHON SOURCE LINES 153-155

Visualize the extracted CNN feature (dimensionality reduction via principle component analysis)


.. GENERATED FROM PYTHON SOURCE LINES 155-176

.. code-block:: default

    pca_dim_reduc = PCAdimReduc(n_components=3, whiten=True)
    feat_dim_reduc = pca_dim_reduc.fit_transform(
        np.concatenate((
            feat1_upsample.transpose((0, 2, 3, 1)).reshape((-1, num_features)).numpy(),
            feat2_upsample.transpose((0, 2, 3, 1)).reshape((-1, num_features)).numpy()
        ), axis=0)
    )
    feat_dim_reduc = feat_dim_reduc / np.max(np.abs(feat_dim_reduc), axis=0, keepdims=True) / 2 + 0.5
    feat1_dim_reduc = feat_dim_reduc[:obj_resize[0] * obj_resize[1], :]
    feat2_dim_reduc = feat_dim_reduc[obj_resize[0] * obj_resize[1]:, :]

    plt.figure(figsize=(8, 4))
    plt.subplot(1, 2, 1)
    plt.title('Image 1 with CNN features')
    plot_image_with_graph(img1, kpts1, A1)
    plt.imshow(feat1_dim_reduc.reshape((obj_resize[1], obj_resize[0], 3)), alpha=0.5)
    plt.subplot(1, 2, 2)
    plt.title('Image 2 with CNN features')
    plot_image_with_graph(img2, kpts2, A2)
    plt.imshow(feat2_dim_reduc.reshape((obj_resize[1], obj_resize[0], 3)), alpha=0.5)




.. image-sg:: /auto_examples/paddle/images/sphx_glr_plot_image_matching_paddle_003.png
   :alt: Image 1 with CNN features, Image 2 with CNN features
   :srcset: /auto_examples/paddle/images/sphx_glr_plot_image_matching_paddle_003.png
   :class: sphx-glr-single-img


.. rst-class:: sphx-glr-script-out

 .. code-block:: none


    <matplotlib.image.AxesImage object at 0x7fa4afe27400>



.. GENERATED FROM PYTHON SOURCE LINES 177-179

Extract node features by nearest interpolation


.. GENERATED FROM PYTHON SOURCE LINES 179-185

.. code-block:: default

    rounded_kpts1 = paddle.cast(paddle.round(kpts1), dtype='int64')
    rounded_kpts2 = paddle.cast(paddle.round(kpts2), dtype='int64')

    node1 = feat1_upsample.transpose((2, 3, 0, 1))[rounded_kpts1[1], rounded_kpts1[0]][:, 0]
    node2 = feat2_upsample.transpose((2, 3, 0, 1))[rounded_kpts2[1], rounded_kpts2[0]][:, 0]








.. GENERATED FROM PYTHON SOURCE LINES 186-197

Build affinity matrix
----------------------
We follow the formulation of Quadratic Assignment Problem (QAP):

.. math::

    &\max_{\mathbf{X}} \ \texttt{vec}(\mathbf{X})^\top \mathbf{K} \texttt{vec}(\mathbf{X})\\
    s.t. \quad &\mathbf{X} \in \{0, 1\}^{n_1\times n_2}, \ \mathbf{X}\mathbf{1} = \mathbf{1}, \ \mathbf{X}^\top\mathbf{1} \leq \mathbf{1}

where the first step is to build the affinity matrix (:math:`\mathbf{K}`)


.. GENERATED FROM PYTHON SOURCE LINES 197-203

.. code-block:: default

    conn1, edge1 = pygm.utils.dense_to_sparse(A1)
    conn2, edge2 = pygm.utils.dense_to_sparse(A2)
    import functools
    gaussian_aff = functools.partial(pygm.utils.gaussian_aff_fn, sigma=1) # set affinity function
    K = pygm.utils.build_aff_mat(node1, edge1, conn1, node2, edge2, conn2, edge_aff_fn=gaussian_aff)








.. GENERATED FROM PYTHON SOURCE LINES 204-210

Visualization of the affinity matrix. For graph matching problem with :math:`N` nodes, the affinity matrix
has :math:`N^2\times N^2` elements because there are :math:`N^2` edges in each graph.

.. note::
    The diagonal elements are node affinities, the off-diagonal elements are edge features.


.. GENERATED FROM PYTHON SOURCE LINES 210-214

.. code-block:: default

    plt.figure(figsize=(4, 4))
    plt.title(f'Affinity Matrix (size: {K.shape[0]}$\\times${K.shape[1]})')
    plt.imshow(K.numpy(), cmap='Blues')




.. image-sg:: /auto_examples/paddle/images/sphx_glr_plot_image_matching_paddle_004.png
   :alt: Affinity Matrix (size: 100$\times$100)
   :srcset: /auto_examples/paddle/images/sphx_glr_plot_image_matching_paddle_004.png
   :class: sphx-glr-single-img


.. rst-class:: sphx-glr-script-out

 .. code-block:: none


    <matplotlib.image.AxesImage object at 0x7fa4afd5c730>



.. GENERATED FROM PYTHON SOURCE LINES 215-219

Solve graph matching problem by RRWM solver
-------------------------------------------
See :func:`~pygmtools.classic_solvers.rrwm` for the API reference.


.. GENERATED FROM PYTHON SOURCE LINES 219-221

.. code-block:: default

    X = pygm.rrwm(K, kpts1.shape[1], kpts2.shape[1])








.. GENERATED FROM PYTHON SOURCE LINES 222-224

The output of RRWM is a soft matching matrix. Hungarian algorithm is then adopted to reach a discrete matching matrix.


.. GENERATED FROM PYTHON SOURCE LINES 224-226

.. code-block:: default

    X = pygm.hungarian(X)








.. GENERATED FROM PYTHON SOURCE LINES 227-232

Plot the matching
------------------
The correct matchings are marked by green, and wrong matchings are marked by red. In this example, the nodes are
ordered by their ground truth classes (i.e. the ground truth matching matrix is a diagonal matrix).


.. GENERATED FROM PYTHON SOURCE LINES 232-244

.. code-block:: default

    plt.figure(figsize=(8, 4))
    plt.suptitle('Image Matching Result by RRWM')
    ax1 = plt.subplot(1, 2, 1)
    plot_image_with_graph(img1, kpts1, A1)
    ax2 = plt.subplot(1, 2, 2)
    plot_image_with_graph(img2, kpts2, A2)
    for i in range(X.shape[0]):
        j = paddle.argmax(X[i]).item()
        con = ConnectionPatch(xyA=kpts1[:, i], xyB=kpts2[:, j], coordsA="data", coordsB="data",
                              axesA=ax1, axesB=ax2, color="red" if i != j else "green")
        plt.gca().add_artist(con)




.. image-sg:: /auto_examples/paddle/images/sphx_glr_plot_image_matching_paddle_005.png
   :alt: Image Matching Result by RRWM
   :srcset: /auto_examples/paddle/images/sphx_glr_plot_image_matching_paddle_005.png
   :class: sphx-glr-single-img





.. GENERATED FROM PYTHON SOURCE LINES 245-253

Solve by other solvers
-----------------------
We could also do a quick benchmarking of other solvers on this specific problem.

IPFP solver
^^^^^^^^^^^
See :func:`~pygmtools.classic_solvers.ipfp` for the API reference.


.. GENERATED FROM PYTHON SOURCE LINES 253-267

.. code-block:: default

    X = pygm.ipfp(K, kpts1.shape[1], kpts2.shape[1])

    plt.figure(figsize=(8, 4))
    plt.suptitle('Image Matching Result by IPFP')
    ax1 = plt.subplot(1, 2, 1)
    plot_image_with_graph(img1, kpts1, A1)
    ax2 = plt.subplot(1, 2, 2)
    plot_image_with_graph(img2, kpts2, A2)
    for i in range(X.shape[0]):
        j = paddle.argmax(X[i]).item()
        con = ConnectionPatch(xyA=kpts1[:, i], xyB=kpts2[:, j], coordsA="data", coordsB="data",
                              axesA=ax1, axesB=ax2, color="red" if i != j else "green")
        plt.gca().add_artist(con)




.. image-sg:: /auto_examples/paddle/images/sphx_glr_plot_image_matching_paddle_006.png
   :alt: Image Matching Result by IPFP
   :srcset: /auto_examples/paddle/images/sphx_glr_plot_image_matching_paddle_006.png
   :class: sphx-glr-single-img





.. GENERATED FROM PYTHON SOURCE LINES 268-272

SM solver
^^^^^^^^^^^
See :func:`~pygmtools.classic_solvers.sm` for the API reference.


.. GENERATED FROM PYTHON SOURCE LINES 272-287

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
        j = paddle.argmax(X[i]).item()
        con = ConnectionPatch(xyA=kpts1[:, i], xyB=kpts2[:, j], coordsA="data", coordsB="data",
                              axesA=ax1, axesB=ax2, color="red" if i != j else "green")
        plt.gca().add_artist(con)




.. image-sg:: /auto_examples/paddle/images/sphx_glr_plot_image_matching_paddle_007.png
   :alt: Image Matching Result by SM
   :srcset: /auto_examples/paddle/images/sphx_glr_plot_image_matching_paddle_007.png
   :class: sphx-glr-single-img





.. GENERATED FROM PYTHON SOURCE LINES 288-299

NGM solver
^^^^^^^^^^^
See :func:`~pygmtools.neural_solvers.ngm` for the API reference.

.. note::
    The NGM solvers are pretrained on a different problem setting, so their performance may seem inferior.
    To improve their performance, you may change the way of building affinity matrices, or try finetuning
    NGM on the new problem.

The NGM solver pretrained on Willow dataset:


.. GENERATED FROM PYTHON SOURCE LINES 299-314

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
        j = paddle.argmax(X[i]).item()
        con = ConnectionPatch(xyA=kpts1[:, i], xyB=kpts2[:, j], coordsA="data", coordsB="data",
                              axesA=ax1, axesB=ax2, color="red" if i != j else "green")
        plt.gca().add_artist(con)




.. image-sg:: /auto_examples/paddle/images/sphx_glr_plot_image_matching_paddle_008.png
   :alt: Image Matching Result by NGM (willow pretrain)
   :srcset: /auto_examples/paddle/images/sphx_glr_plot_image_matching_paddle_008.png
   :class: sphx-glr-single-img


.. rst-class:: sphx-glr-script-out

 .. code-block:: none


    Downloading to /Users/guoziao/Library/Caches/pygmtools/ngm_willow_paddle.pdparams...

    Downloading to /Users/guoziao/Library/Caches/pygmtools/ngm_willow_paddle.pdparams...
    Warning: Network error. Retrying...
     HTTPSConnectionPool(host='doc-14-9c-docs.googleusercontent.com', port=443): Max retries exceeded with url: /docs/securesc/ha0ro937gcuc7l7deffksulhg5h7mbp1/k148r0af8592f4igvstdlausa5eisbm9/1678177650000/01336986364963931990/*/1j1kXWsassE3bAVWjPy2g0jUGG2IeODc8?e=download&uuid=b3154626-202b-47fb-81b1-5c1ad56da294 (Caused by ProxyError('Cannot connect to proxy.', OSError(0, 'Error')))

    Downloading to /Users/guoziao/Library/Caches/pygmtools/ngm_willow_paddle.pdparams...

    Downloading to /Users/guoziao/Library/Caches/pygmtools/ngm_willow_paddle.pdparams...
    Warning: Network error. Retrying...
     HTTPSConnectionPool(host='drive.google.com', port=443): Max retries exceeded with url: /u/0/uc?export=download&confirm=Z-AR&id=1j1kXWsassE3bAVWjPy2g0jUGG2IeODc8 (Caused by ProxyError('Cannot connect to proxy.', OSError(0, 'Error')))

    Downloading to /Users/guoziao/Library/Caches/pygmtools/ngm_willow_paddle.pdparams...

    Downloading to /Users/guoziao/Library/Caches/pygmtools/ngm_willow_paddle.pdparams...
    Warning: Network error. Retrying...
     HTTPSConnectionPool(host='drive.google.com', port=443): Max retries exceeded with url: /u/0/uc?export=download&confirm=Z-AR&id=1j1kXWsassE3bAVWjPy2g0jUGG2IeODc8 (Caused by ProxyError('Cannot connect to proxy.', OSError(0, 'Error')))

    Downloading to /Users/guoziao/Library/Caches/pygmtools/ngm_willow_paddle.pdparams...

    Downloading to /Users/guoziao/Library/Caches/pygmtools/ngm_willow_paddle.pdparams...
    Warning: Network error. Retrying...
     HTTPSConnectionPool(host='doc-14-9c-docs.googleusercontent.com', port=443): Max retries exceeded with url: /docs/securesc/ha0ro937gcuc7l7deffksulhg5h7mbp1/m9ee6124r22sbpmhhmhjspduurjittoi/1678177725000/01336986364963931990/*/1j1kXWsassE3bAVWjPy2g0jUGG2IeODc8?e=download&uuid=c1609b68-5a1e-4113-a327-0e841cd19a3f (Caused by ProxyError('Cannot connect to proxy.', OSError(0, 'Error')))

    Downloading to /Users/guoziao/Library/Caches/pygmtools/ngm_willow_paddle.pdparams...

    Downloading to /Users/guoziao/Library/Caches/pygmtools/ngm_willow_paddle.pdparams...
    Warning: Network error. Retrying...
     HTTPSConnectionPool(host='drive.google.com', port=443): Max retries exceeded with url: /u/0/uc?export=download&confirm=Z-AR&id=1j1kXWsassE3bAVWjPy2g0jUGG2IeODc8 (Caused by ProxyError('Cannot connect to proxy.', OSError(0, 'Error')))




.. GENERATED FROM PYTHON SOURCE LINES 315-317

The NGM solver pretrained on VOC dataset:


.. GENERATED FROM PYTHON SOURCE LINES 317-330

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
        j = paddle.argmax(X[i]).item()
        con = ConnectionPatch(xyA=kpts1[:, i], xyB=kpts2[:, j], coordsA="data", coordsB="data",
                              axesA=ax1, axesB=ax2, color="red" if i != j else "green")
        plt.gca().add_artist(con)


.. image-sg:: /auto_examples/paddle/images/sphx_glr_plot_image_matching_paddle_009.png
   :alt: Image Matching Result by NGM (voc pretrain)
   :srcset: /auto_examples/paddle/images/sphx_glr_plot_image_matching_paddle_009.png
   :class: sphx-glr-single-img






.. rst-class:: sphx-glr-timing

   **Total running time of the script:** ( 0 minutes  59.448 seconds)


.. _sphx_glr_download_auto_examples_paddle_plot_image_matching_paddle.py:

.. only:: html

  .. container:: sphx-glr-footer sphx-glr-footer-example


    .. container:: sphx-glr-download sphx-glr-download-python

      :download:`Download Python source code: plot_image_matching_paddle.py <plot_image_matching_paddle.py>`

    .. container:: sphx-glr-download sphx-glr-download-jupyter

      :download:`Download Jupyter notebook: plot_image_matching_paddle.ipynb <plot_image_matching_paddle.ipynb>`


.. only:: html

 .. rst-class:: sphx-glr-signature

    `Gallery generated by Sphinx-Gallery <https://sphinx-gallery.github.io>`_
