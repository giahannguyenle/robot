#!/usr/bin/env python3

import rclpy
from rclpy.node import Node
from sensor_msgs.msg import Image
from cv_bridge import CvBridge
import cv2

class CsiCameraReaderNode(Node):
    def __init__(self):
        super().__init__('csi_camera_reader_node')
        self.publisher_ = self.create_publisher(Image, 'camera/image_raw', 10)
        timer_period = 0.1  # 10 FPS
        self.timer = self.create_timer(timer_period, self.timer_callback)
        self.bridge = CvBridge()

        # GStreamer pipeline for Jetson Nano CSI camera
        self.cap = cv2.VideoCapture(self.gstreamer_pipeline(), cv2.CAP_GSTREAMER)
        if not self.cap.isOpened():
            self.get_logger().error('Failed to open CSI camera.')
        else:
            self.get_logger().info('CSI camera opened successfully.')

    def gstreamer_pipeline(self, capture_width=1280, capture_height=720, display_width=1280,
                           display_height=720, framerate=30, flip_method=0):
        return (
            f'nvarguscamerasrc ! '
            f'video/x-raw(memory:NVMM), width={capture_width}, height={capture_height}, '
            f'format=NV12, framerate={framerate}/1 ! '
            f'nvvidconv flip-method={flip_method} ! '
            f'video/x-raw, width={display_width}, height={display_height}, format=BGRx ! '
            f'videoconvert ! video/x-raw, format=BGR ! appsink'
        )

    def timer_callback(self):
        ret, frame = self.cap.read()
        if ret:
            msg = self.bridge.cv2_to_imgmsg(frame, encoding='bgr8')
            self.publisher_.publish(msg)
        else:
            self.get_logger().warn('No frame received from camera.')

    def destroy_node(self):
        if self.cap.isOpened():
            self.cap.release()
        super().destroy_node()

def main(args=None):
    rclpy.init(args=args)
    node = CsiCameraReaderNode()
    try:
        rclpy.spin(node)
    except KeyboardInterrupt:
        pass
    node.destroy_node()
    rclpy.shutdown()
