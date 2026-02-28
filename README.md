
"""
CamDraw - Color Tracking Drawing App
=====================================
Draw with a RED object in front of your camera.
Supports front/back camera switching.
Works on Android, iOS, and Desktop (Kivy).

SETUP:
    pip install kivy numpy

RUN:
    python cam_draw.py

CALIBRATION:
    If the camera preview is not aligned with your drawing:
    1. Adjust ROTATION values (0–3) for each camera.
    2. Toggle FLIP_H / FLIP_V if the drawing is mirrored.
    3. Adjust color thresholds for your object.
"""

# ─────────────────────────────────────────────────────────────────────────────
#  USER CONFIGURATION
# ─────────────────────────────────────────────────────────────────────────────

# Camera settings: index -> {rotation, flip_h, flip_v, label}
# rotation: 0=0°, 1=90° CW, 2=180°, 3=270° CW
CAMERA_CONFIGS = {
    1: dict(rotation=1, flip_h=False, flip_v=False, label="Back"),
    0: dict(rotation=1, flip_h=True,  flip_v=False, label="Front"),
}

# Color thresholds for red object detection
COLOR_R_MIN = 140
COLOR_G_MAX = 90
COLOR_B_MAX = 90

# Performance
SAMPLE_STEP = 6       # Lower = more precise but slower
FPS = 30              # Frames per second

# Drawing style
DRAW_COLOR      = (0.1, 0.95, 0.4, 1.0)   # Bright green
DRAW_LINE_WIDTH = 3

# ─────────────────────────────────────────────────────────────────────────────
#  IMPORTS
# ─────────────────────────────────────────────────────────────────────────────

import numpy as np

from kivy.app          import App
from kivy.clock        import Clock
from kivy.core.window  import Window
from kivy.graphics     import Color, Line, Rectangle
from kivy.graphics.texture import Texture
from kivy.uix.camera   import Camera
from kivy.uix.floatlayout import FloatLayout
from kivy.uix.image    import Image
from kivy.uix.label    import Label
from kivy.uix.button   import Button
from kivy.uix.widget   import Widget

# ─────────────────────────────────────────────────────────────────────────────
#  DRAWING LAYER
# ─────────────────────────────────────────────────────────────────────────────

class DrawLayer(Widget):
    """Transparent widget that accumulates drawn lines."""

    def __init__(self, **kwargs):
        super().__init__(**kwargs)
        self.active_line = None
        self.size_hint = (1, 1)

    def start_stroke(self, x, y):
        with self.canvas:
            Color(*DRAW_COLOR)
            self.active_line = Line(points=[x, y], width=DRAW_LINE_WIDTH,
                                    cap='round', joint='round')

    def extend_stroke(self, x, y):
        if self.active_line is None:
            self.start_stroke(x, y)
        else:
            self.active_line.points += [x, y]

    def break_stroke(self):
        self.active_line = None

    def clear_all(self):
        self.canvas.clear()
        self.active_line = None

# ─────────────────────────────────────────────────────────────────────────────
#  MAIN LAYOUT
# ─────────────────────────────────────────────────────────────────────────────

class CamDraw(FloatLayout):

    def __init__(self, **kwargs):
        super().__init__(**kwargs)

        self.camera_index = 0
        self.prev_detected = False

        # Hidden camera widget
        self._camera = self._build_camera(self.camera_index)
        self.add_widget(self._camera)

        # Full-screen video display (fills entire layout)
        self.display = Image(
            size_hint=(1, 1),
            pos_hint={'x': 0, 'y': 0},
            allow_stretch=True,
            keep_ratio=False,          # Fill exactly, may distort
        )
        self.add_widget(self.display)

        # Drawing layer (on top of video)
        self.draw_layer = DrawLayer(size=self.size, pos=self.pos)
        self.bind(size=self._sync_draw_layer, pos=self._sync_draw_layer)
        self.add_widget(self.draw_layer)

        # Controls overlay
        self._build_hud()

        # Start frame processing
        Clock.schedule_interval(self._process_frame, 1.0 / FPS)

    # -------------------------------------------------------------------------
    # Helpers
    # -------------------------------------------------------------------------
    def _sync_draw_layer(self, *_):
        self.draw_layer.size = self.size
        self.draw_layer.pos = self.pos

    def _build_camera(self, index):
        cam = Camera(play=True, index=index, resolution=(640, 480))
        cam.opacity = 0
        cam.size_hint = (0.01, 0.01)
        cam.pos = (-9999, -9999)
        return cam

    def _build_hud(self):
        # Status label – top center
        self.status = Label(
            text="Point a RED object at the camera",
            size_hint=(0.6, 0.07),
            pos_hint={'center_x': 0.5, 'top': 1},
            color=(1, 1, 1, 0.95),
            font_size='15sp',
            halign='center',
            valign='middle',
        )
        self.add_widget(self.status)

        # Semi-transparent bottom bar
        bar = Widget(size_hint=(1, 0.1), pos_hint={'x': 0, 'y': 0})
        with bar.canvas.before:
            Color(0, 0, 0, 0.55)
            self._bar_rect = Rectangle(size=bar.size, pos=bar.pos)
        bar.bind(size=lambda w, v: setattr(self._bar_rect, 'size', v),
                 pos=lambda w, v: setattr(self._bar_rect, 'pos', v))
        self.add_widget(bar)

        # Switch camera button – bottom left
        sw = Button(
            text="⟳ Camera",
            size_hint=(0.3, 0.08),
            pos_hint={'x': 0.03, 'y': 0.01},
            background_color=(0.15, 0.5, 1, 1),
            color=(1, 1, 1, 1),
            font_size='14sp',
            bold=True,
        )
        sw.bind(on_press=self._toggle_camera)
        self.add_widget(sw)

        # Clear button – bottom right
        cl = Button(
            text="✕ Clear",
            size_hint=(0.3, 0.08),
            pos_hint={'right': 0.97, 'y': 0.01},
            background_color=(0.9, 0.2, 0.2, 1),
            color=(1, 1, 1, 1),
            font_size='14sp',
            bold=True,
        )
        cl.bind(on_press=lambda *_: self.draw_layer.clear_all())
        self.add_widget(cl)

        # Camera label – bottom center
        self.cam_label = Label(
            text=self._cam_name(),
            size_hint=(0.3, 0.08),
            pos_hint={'center_x': 0.5, 'y': 0.01},
            color=(0.7, 0.7, 0.7, 1),
            font_size='13sp',
        )
        self.add_widget(self.cam_label)

    # -------------------------------------------------------------------------
    # Camera switching
    # -------------------------------------------------------------------------
    def _cam_name(self):
        cfg = CAMERA_CONFIGS.get(self.camera_index, {})
        return cfg.get('label', f'Camera {self.camera_index}')

    def _toggle_camera(self, *_):
        self._camera.play = False
        self.remove_widget(self._camera)

        indices = sorted(CAMERA_CONFIGS.keys())
        idx = indices.index(self.camera_index)
        self.camera_index = indices[(idx + 1) % len(indices)]

        self._camera = self._build_camera(self.camera_index)
        self.add_widget(self._camera, index=len(self.children))  # behind everything

        self.draw_layer.break_stroke()
        self.cam_label.text = self._cam_name()
        self.status.text = f"Switched to {self._cam_name()} camera"

    # -------------------------------------------------------------------------
    # Frame processing
    # -------------------------------------------------------------------------
    def _get_cfg(self):
        return CAMERA_CONFIGS.get(self.camera_index,
                                   dict(rotation=0, flip_h=False, flip_v=False))

    def _apply_transform(self, frame, cfg):
        """Rotate and flip a numpy RGBA frame."""
        rot = cfg.get('rotation', 0)
        if rot == 1:
            frame = np.rot90(frame, k=-1)   # 90° CW
        elif rot == 2:
            frame = np.rot90(frame, k=2)    # 180°
        elif rot == 3:
            frame = np.rot90(frame, k=1)    # 270° CW
        if cfg.get('flip_h'):
            frame = np.fliplr(frame)
        if cfg.get('flip_v'):
            frame = np.flipud(frame)
        return frame

    def _process_frame(self, dt):
        tex = self._camera.texture
        if tex is None:
            return

        tw, th = tex.size
        raw = tex.pixels
        if not raw or len(raw) < tw * th * 4:
            return

        try:
            frame = np.frombuffer(raw, dtype=np.uint8).reshape((th, tw, 4))
        except ValueError:
            return

        # Apply orientation
        cfg = self._get_cfg()
        frame = self._apply_transform(frame, cfg)
        rh, rw = frame.shape[:2]

        # Update display texture
        new_tex = Texture.create(size=(rw, rh), colorfmt='rgba')
        new_tex.blit_buffer(frame.tobytes(), colorfmt='rgba', bufferfmt='ubyte')
        new_tex.flip_vertical()        # Kivy Y-axis correction
        self.display.texture = new_tex

        # --- Color tracking ---
        # Downsample
        s = frame[::SAMPLE_STEP, ::SAMPLE_STEP, :]
        r = s[:, :, 0].astype(np.int16)
        g = s[:, :, 1].astype(np.int16)
        b = s[:, :, 2].astype(np.int16)
        mask = (r > COLOR_R_MIN) & (g < COLOR_G_MAX) & (b < COLOR_B_MAX)
        ys, xs = np.where(mask)

        if len(xs) == 0:
            if self.prev_detected:
                self.draw_layer.break_stroke()
                self.prev_detected = False
            self.status.text = "No target detected – show a RED object"
            return

        # Centroid in full-resolution frame
        cx = float(np.mean(xs)) * SAMPLE_STEP
        cy = float(np.mean(ys)) * SAMPLE_STEP

        # Normalized coordinates (0-1) in camera frame
        norm_x = cx / rw
        norm_y = cy / rh

        # Map to display widget coordinates
        # The display fills the entire layout, so we use its size and position
        disp = self.display
        sx = disp.x + norm_x * disp.width
        sy = disp.y + (1 - norm_y) * disp.height   # flip Y

        # Draw
        if self.prev_detected:
            self.draw_layer.extend_stroke(sx, sy)
        else:
            self.draw_layer.start_stroke(sx, sy)
        self.prev_detected = True

        count = len(xs)
        self.status.text = f"Tracking ({int(sx)}, {int(sy)}) – {count * SAMPLE_STEP**2} px"

# ─────────────────────────────────────────────────────────────────────────────
#  APP
# ─────────────────────────────────────────────────────────────────────────────

class CamDrawApp(App):
    title = "CamDraw"

    def build(self):
        Window.clearcolor = (0.05, 0.05, 0.05, 1)
        return CamDraw()

    def on_stop(self):
        if hasattr(self.root, '_camera'):
            self.root._camera.play = False

# ─────────────────────────────────────────────────────────────────────────────

if __name__ == '__main__':
    CamDrawApp().run()
