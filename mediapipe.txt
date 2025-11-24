import cv2                 # Thư viện xử lý ảnh và camera
import mediapipe as mp     # Thư viện nhận diện bàn tay
import pyautogui           # Thư viện mô phỏng nhấn phím (WASD)
import win32gui            # Điều khiển cửa sổ Windows (Always on top)
import win32con            # Các hằng số cho win32gui
import time                # Dùng để tạm dừng (sleep)


class HandControl:
    def __init__(self):
        self.mp_hands = mp.solutions.hands  # Module hands của Mediapipe
        # Tạo đối tượng nhận diện bàn tay với độ tin cậy ≥ 0.7
        self.hands = self.mp_hands.Hands(min_detection_confidence=0.7,
                                         min_tracking_confidence=0.7)
        self.mp_draw = mp.solutions.drawing_utils  # Công cụ vẽ bàn tay
        self.current_zone = None                   # Lưu ô hiện tại (1–9)
        self.window_name = "Hand WASD 3x3 - Middle Finger Right Hand"  # Tên cửa sổ

    def get_zone(self, x_norm, y_norm):
        col = int(x_norm * 3)              # Cột: 0,1,2
        row = int(y_norm * 3)              # Hàng: 0,1,2
        zone = row * 3 + col + 1           # Zone 1–9
        return zone

    def control(self):
        cap = cv2.VideoCapture(0)          # Mở camera
        cap.set(3, 640)                    # Chiều rộng camera
        cap.set(4, 480)                    # Chiều cao camera

        cv2.namedWindow(self.window_name, cv2.WINDOW_NORMAL)  # Tạo cửa sổ

        time.sleep(0.5)                    # Chờ OpenCV tạo xong cửa sổ
        hwnd = win32gui.FindWindow(None, self.window_name)  # Lấy handle cửa sổ

        # Đặt Always on Top tại vị trí (100,100) với kích thước 640x480
        win32gui.SetWindowPos(hwnd, win32con.HWND_TOPMOST,
                              100, 100, 640, 480,
                              win32con.SWP_SHOWWINDOW)

        while True:
            ret, frame = cap.read()        # Đọc frame từ camera
            if not ret:                    # Nếu lỗi -> thoát
                break

            frame = cv2.flip(frame, 1)     # Lật ảnh ngang (giống gương)
            h, w, _ = frame.shape

            img_rgb = cv2.cvtColor(frame, cv2.COLOR_BGR2RGB)  # Chuyển BGR -> RGB
            results = self.hands.process(img_rgb)             # Nhận diện bàn tay

            if results.multi_hand_landmarks:                  # Nếu phát hiện bàn tay
                for handLms, handedness in zip(results.multi_hand_landmarks, results.multi_handedness):
                    label = handedness.classification[0].label  # Lấy nhãn: "Left" hoặc "Right"
                    if label == "Right":                        # Chỉ xử lý tay phải
                        x = handLms.landmark[12].x              # Ngón giữa (landmark 12)
                        y = handLms.landmark[12].y
                        new_zone = self.get_zone(x, y)          # Xác định zone 1–9

                        if new_zone != self.current_zone:       # Nếu đổi sang ô khác
                            self.current_zone = new_zone        # Cập nhật zone
                            if new_zone == 2:                   # Ô trên giữa
                                pyautogui.press('w')
                                print("W pressed")
                            elif new_zone == 4:                 # Ô giữa trái
                                pyautogui.press('a')
                                print("A pressed")
                            elif new_zone == 6:                 # Ô giữa phải
                                pyautogui.press('d')
                                print("D pressed")
                            elif new_zone == 8:                 # Ô dưới giữa
                                pyautogui.press('s')
                                print("S pressed")
                            else:
                                print("No action")

                        # Vẽ bàn tay + khớp
                        self.mp_draw.draw_landmarks(frame, handLms, self.mp_hands.HAND_CONNECTIONS)
            else:
                self.current_zone = None   # Nếu không thấy bàn tay -> reset

            # Vẽ lưới 3x3
            cv2.line(frame, (w // 3, 0), (w // 3, h), (0, 255, 0), 2)
            cv2.line(frame, (2 * w // 3, 0), (2 * w // 3, h), (0, 255, 0), 2)
            cv2.line(frame, (0, h // 3), (w, h // 3), (0, 255, 0), 2)
            cv2.line(frame, (0, 2 * h // 3), (w, 2 * h // 3), (0, 255, 0), 2)

            cv2.imshow(self.window_name, frame)  # Hiển thị camera

            if cv2.waitKey(1) & 0xFF == ord('q'):  # Nhấn Q để thoát
                break

        cap.release()                     # Giải phóng camera
        cv2.destroyAllWindows()           # Đóng cửa sổ


if __name__ == "__main__":
    HandControl().control()
