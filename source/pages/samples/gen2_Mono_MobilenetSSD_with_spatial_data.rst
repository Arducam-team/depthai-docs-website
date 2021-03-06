Gen2 在双目相机上运行MobilenetSSD神经网络并获取深度信息
============================================================

本示例说明如何在校正后的右侧输入帧上运行MobileNetv2SSD，以及如何显示预览，检测，深度图和空间信息（X，Y，Z）。除了具有空间数据外，X，Y，Z坐标相对于深度图的中心。

setConfidenceThreshold-置信度阈值，高于该阈值将检测到对象

演示
**************


设置
********************

.. warning::
    说明：此处安装的是第二代depthai库

请运行以下命令来安装所需的依赖项

.. code-block:: bash

    python3 -m pip install -U pip
    python3 -m pip install opencv-python
    python3 -m pip install -U --force-reinstall depthai

有关更多信息，请参阅 :ref:`Python API 安装指南 <Python API安装详解>`

这个示例还需要 mobileenetsdd blob ( :code:`mobilenet.blob` 文件 )才能工作——您可以从 `这里 <https://artifacts.luxonis.com/artifactory/luxonis-depthai-data-local/network/mobilenet-ssd_openvino_2021.2_6shave.blob>`_ 下载它。


源代码
*********************

可以在 `GitHub <https://github.com/luxonis/depthai-python/blob/main/examples/26_2_spatial_mobilenet_mono.py>`_ 上找到。国内用户也可以在 `gitee <https://gitee.com/oakchina/depthai-python/blob/main/examples/26_2_spatial_mobilenet_mono.py>`_ 上找到。

.. code-block:: python

    #!/usr/bin/env python3

    from pathlib import Path
    import sys
    import cv2
    import depthai as dai
    import numpy as np
    import time

    '''
    Mobilenet SSD设备端解码演示
     “ mobilenet-ssd”模型是旨在实现单发多框检测（SSD）的网络
     执行对象检测。 该模型是使用Caffe *框架实现的。
     有关此模型的详细信息，请查看存储库<https://github.com/chuanqi305/MobileNet-SSD>。
    '''

    # MobilenetSSD标签
    labelMap = ["background", "aeroplane", "bicycle", "bird", "boat", "bottle", "bus", "car", "cat", "chair", "cow",
                "diningtable", "dog", "horse", "motorbike", "person", "pottedplant", "sheep", "sofa", "train", "tvmonitor"]

    syncNN = True
    flipRectified = True

    # 首先获取模型
    nnPath = str((Path(__file__).parent / Path('models/mobilenet-ssd_openvino_2021.2_6shave.blob')).resolve().absolute())
    if len(sys.argv) > 1:
        nnPath = sys.argv[1]

    if not Path(nnPath).exists():
        import sys
        raise FileNotFoundError(f'Required file/s not found, please run "{sys.executable} install_requirements.py"')

    # Start defining a pipeline
    pipeline = dai.Pipeline()


    manip = pipeline.createImageManip()
    manip.initialConfig.setResize(300, 300)
    # NN模型需要BGR输入。 默认情况下，ImageManip输出类型将与输入相同（在这种情况下为灰色）
    manip.initialConfig.setFrameType(dai.ImgFrame.Type.BGR888p)
    # manip.setKeepAspectRatio(False)

    # 定义一个神经网络，它将基于源帧进行预测
    spatialDetectionNetwork = pipeline.createMobileNetSpatialDetectionNetwork()
    spatialDetectionNetwork.setConfidenceThreshold(0.5)
    spatialDetectionNetwork.setBlobPath(nnPath)
    spatialDetectionNetwork.input.setBlocking(False)
    spatialDetectionNetwork.setBoundingBoxScaleFactor(0.5)
    spatialDetectionNetwork.setDepthLowerThreshold(100)
    spatialDetectionNetwork.setDepthUpperThreshold(5000)

    manip.out.link(spatialDetectionNetwork.input)

    # 创建输出流
    xoutManip = pipeline.createXLinkOut()
    xoutManip.setStreamName("right")
    if syncNN:
        spatialDetectionNetwork.passthrough.link(xoutManip.input)
    else:
        manip.out.link(xoutManip.input)

    depthRoiMap = pipeline.createXLinkOut()
    depthRoiMap.setStreamName("boundingBoxDepthMapping")

    xoutDepth = pipeline.createXLinkOut()
    xoutDepth.setStreamName("depth")

    nnOut = pipeline.createXLinkOut()
    nnOut.setStreamName("detections")
    spatialDetectionNetwork.out.link(nnOut.input)
    spatialDetectionNetwork.boundingBoxMapping.link(depthRoiMap.input)

    monoLeft = pipeline.createMonoCamera()
    monoRight = pipeline.createMonoCamera()
    stereo = pipeline.createStereoDepth()
    monoLeft.setResolution(dai.MonoCameraProperties.SensorResolution.THE_400_P)
    monoLeft.setBoardSocket(dai.CameraBoardSocket.LEFT)
    monoRight.setResolution(dai.MonoCameraProperties.SensorResolution.THE_400_P)
    monoRight.setBoardSocket(dai.CameraBoardSocket.RIGHT)
    stereo.setConfidenceThreshold(255)

    stereo.rectifiedRight.link(manip.inputImage)

    monoLeft.out.link(stereo.left)
    monoRight.out.link(stereo.right)

    stereo.depth.link(spatialDetectionNetwork.inputDepth)
    spatialDetectionNetwork.passthroughDepth.link(xoutDepth.input)

    # 连接并启动管道
    with dai.Device(pipeline) as device:

        # 输出队列将用于从上面定义的输出中获取rgb帧和nn数据
        previewQueue = device.getOutputQueue(name="right", maxSize=4, blocking=False)
        detectionNNQueue = device.getOutputQueue(name="detections", maxSize=4, blocking=False)
        depthRoiMap = device.getOutputQueue(name="boundingBoxDepthMapping", maxSize=4, blocking=False)
        depthQueue = device.getOutputQueue(name="depth", maxSize=4, blocking=False)

        rectifiedRight = None
        detections = []

        startTime = time.monotonic()
        counter = 0
        fps = 0
        color = (255, 255, 255)

        while True:
            inRectified = previewQueue.get()
            det = detectionNNQueue.get()
            depth = depthQueue.get()

            counter += 1
            currentTime = time.monotonic()
            if (currentTime - startTime) > 1:
                fps = counter / (currentTime - startTime)
                counter = 0
                startTime = currentTime

            rectifiedRight = inRectified.getCvFrame()

            depthFrame = depth.getFrame()

            depthFrameColor = cv2.normalize(depthFrame, None, 255, 0, cv2.NORM_INF, cv2.CV_8UC1)
            depthFrameColor = cv2.equalizeHist(depthFrameColor)
            depthFrameColor = cv2.applyColorMap(depthFrameColor, cv2.COLORMAP_HOT)
            detections = det.detections
            if len(detections) != 0:
                boundingBoxMapping = depthRoiMap.get()
                roiDatas = boundingBoxMapping.getConfigData()

                for roiData in roiDatas:
                    roi = roiData.roi
                    roi = roi.denormalize(depthFrameColor.shape[1], depthFrameColor.shape[0])
                    topLeft = roi.topLeft()
                    bottomRight = roi.bottomRight()
                    xmin = int(topLeft.x)
                    ymin = int(topLeft.y)
                    xmax = int(bottomRight.x)
                    ymax = int(bottomRight.y)
                    cv2.rectangle(depthFrameColor, (xmin, ymin), (xmax, ymax), color, cv2.FONT_HERSHEY_SCRIPT_SIMPLEX)

            if flipRectified:
                rectifiedRight = cv2.flip(rectifiedRight, 1)

            # 如果rectifiedRight可用，请在其上绘制边界框并显示rectifiedRight
            height = rectifiedRight.shape[0]
            width = rectifiedRight.shape[1]
            for detection in detections:
                if flipRectified:
                    swap = detection.xmin
                    detection.xmin = 1 - detection.xmax
                    detection.xmax = 1 - swap
                # 归一化边界框
                x1 = int(detection.xmin * width)
                x2 = int(detection.xmax * width)
                y1 = int(detection.ymin * height)
                y2 = int(detection.ymax * height)

                try:
                    label = labelMap[detection.label]
                except:
                    label = detection.label

                cv2.putText(rectifiedRight, str(label), (x1 + 10, y1 + 20), cv2.FONT_HERSHEY_TRIPLEX, 0.5, color)
                cv2.putText(rectifiedRight, "{:.2f}".format(detection.confidence*100), (x1 + 10, y1 + 35), cv2.FONT_HERSHEY_TRIPLEX, 0.5, color)
                cv2.putText(rectifiedRight, f"X: {int(detection.spatialCoordinates.x)} mm", (x1 + 10, y1 + 50), cv2.FONT_HERSHEY_TRIPLEX, 0.5, color)
                cv2.putText(rectifiedRight, f"Y: {int(detection.spatialCoordinates.y)} mm", (x1 + 10, y1 + 65), cv2.FONT_HERSHEY_TRIPLEX, 0.5, color)
                cv2.putText(rectifiedRight, f"Z: {int(detection.spatialCoordinates.z)} mm", (x1 + 10, y1 + 80), cv2.FONT_HERSHEY_TRIPLEX, 0.5, color)

                cv2.rectangle(rectifiedRight, (x1, y1), (x2, y2), color, cv2.FONT_HERSHEY_SIMPLEX)

            cv2.putText(rectifiedRight, "NN fps: {:.2f}".format(fps), (2, rectifiedRight.shape[0] - 4), cv2.FONT_HERSHEY_TRIPLEX, 0.4, color)
            cv2.imshow("depth", depthFrameColor)
            cv2.imshow("rectified right", rectifiedRight)

            if cv2.waitKey(1) == ord('q'):
                break


.. include::  /pages/includes/footer-short.rst