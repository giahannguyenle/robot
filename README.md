 import rclpy
from rclpy.node import Node
from sensor_msgs.msg import Image
import cv2
from cv_bridge import CvBridge

class CameraReaderNode(Node):
    def __init__(self):
        super().__init__('camera_loader_node')
        self.bridge = CvBridge()
        self.cap = cv2.VideoCapture('/home/nlgh/Downloads/demo1.mp4')

        if not self.cap.isOpened():
            self.get_logger().error("Failed to open camera")
            self.camera_available = False
            return

        self.camera_available = True
        self.publisher = self.create_publisher(Image, 'camera/image_raw', 10)
        self.timer = self.create_timer(0.1, self.timer_callback)  # 10 Hz

    def timer_callback(self):
        ret, frame = self.cap.read()
        if ret:
            msg = self.bridge.cv2_to_imgmsg(frame, encoding='bgr8')
            self.publisher.publish(msg)
        else:
            self.get_logger().warning("Failed to capture frame from camera")

    def destroy_node(self):
        if hasattr(self, 'cap') and self.cap.isOpened():
            self.cap.release()
        super().destroy_node()

def main(args=None):
    rclpy.init(args=args)
    node = CameraReaderNode()

    if not getattr(node, 'camera_available', False):
        node.destroy_node()
        rclpy.shutdown()
        return

    try:
        rclpy.spin(node)
    except KeyboardInterrupt:
        pass
    finally:
        node.destroy_node()
        rclpy.shutdown()

if __name__ == '__main__':
    main()
