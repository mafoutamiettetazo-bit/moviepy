# make_visualizer_flash.py
# Visualizer rap carré 1080x1080 avec flashs au refrain

import numpy as np
from pathlib import Path
from moviepy.editor import (
    AudioFileClip, ImageClip, CompositeVideoClip, TextClip, vfx
)

# ==========
# Réglages
# ==========
AUDIO_PATH = Path("Trone.mp3")        # mets ton fichier ici (renommé sans accent si possible)
IMAGE_PATH = Path("storyboard.png")   # storyboard

# Sortie vidéo
W, H = 1080, 1080       # format carré
FPS = 24
OUTPUT = Path("Visualizer_Trone_Flash.mp4")

# Infos titre
SHOW_TITLE = True
ARTIST = "TON BLAZE"
TRACK = "TRÔNE"

# Zoom & mouvement
BASE_ZOOM_START = 1.02
BASE_ZOOM_END = 1.08
AUDIO_PULSE_ZOOM = 0.03

# Refrains (en secondes, à ajuster selon ton son)
REFRAINS = [(30, 45), (90, 105)]  
# Exemple : refrain 1 de 30s à 45s, refrain 2 de 90s à 105s

# ==========
# Fonctions
# ==========
def compute_audio_envelope(audio: AudioFileClip, fps: int = 15) -> np.ndarray:
    arr = audio.to_soundarray(fps=11025)
    if arr.ndim == 2:
        arr = arr.mean(axis=1)
    hop = int(11025 / fps)
    env = [np.sqrt((arr[i:i+hop]**2).mean() + 1e-12) for i in range(0, len(arr), hop)]
    env = np.array(env)
    if env.max() > 0:
        env = env / env.max()
    return env

def env_at(t: float, env: np.ndarray, fps_env: int, duration: float) -> float:
    idx = int(np.clip(t * fps_env, 0, len(env) - 1))
    return float(env[idx])

# ==========
# Script principal
# ==========
def main():
    audio = AudioFileClip(str(AUDIO_PATH))
    duration = audio.duration

    # Charger image
    img = ImageClip(str(IMAGE_PATH)).set_duration(duration)
    img = img.resize(height=H) if img.h < H else img.resize(width=W)

    # Enveloppe audio
    FPS_ENV = 15
    envelope = compute_audio_envelope(audio, fps=FPS_ENV)

    # Zoom dynamique
    def dynamic_resize(t):
        base_zoom = BASE_ZOOM_START + (BASE_ZOOM_END - BASE_ZOOM_START) * (t / duration)
        pulse = env_at(t, envelope, FPS_ENV, duration) * AUDIO_PULSE_ZOOM
        return base_zoom + pulse

    moving = (
        img
        .resize(lambda t: dynamic_resize(t))
        .set_position(("center", "center"))
    )

    clips = [moving]

    # Ajout titre
    if SHOW_TITLE:
        title_clip = (
            TextClip(ARTIST, fontsize=60, color="white", font="Arial-Bold", method="label")
            .set_position(("center", H*0.08))
            .set_duration(duration)
            .fadein(0.5).fadeout(0.5)
        )
        track_clip = (
            TextClip(TRACK, fontsize=48, color="white", font="Arial-Bold", method="label")
            .set_position(("center", H*0.16))
            .set_duration(duration)
            .fadein(0.5).fadeout(0.5)
        )
        clips += [title_clip, track_clip]

    # Flashs au refrain
    for start, end in REFRAINS:
        flash = (
            ImageClip(np.ones((H, W, 3)) * 255)  # écran blanc
            .set_start(start).set_duration(end - start)
            .set_opacity(lambda t: 0.8 * abs(np.sin(10 * np.pi * t)))  # clignote
        )
        clips.append(flash)

    final = CompositeVideoClip(clips, size=(W, H)).set_audio(audio)
    final.write_videofile(str(OUTPUT), fps=FPS, codec="libx264", audio_codec="aac")

if _name_ == "_main_":
    main()
