"""
Shoot the Duck (Pygame)
- Move crosshair with mouse
- Left click to shoot ducks
- R to reload instantly (optional), P to pause
- ESC to quit
No external assets required (drawn with shapes).
"""

import math
import random
import sys
from dataclasses import dataclass

import pygame


# -----------------------------
# Config
# -----------------------------
WIDTH, HEIGHT = 1000, 600
FPS = 60

BG_COLOR = (135, 206, 235)       # sky
GROUND_COLOR = (70, 180, 90)     # grass
HUD_COLOR = (15, 15, 15)

CROSSHAIR_COLOR = (30, 30, 30)
SHOT_COLOR = (255, 80, 80)

DUCK_BODY = (30, 140, 60)
DUCK_WING = (20, 110, 50)
DUCK_BEAK = (245, 190, 40)
DUCK_EYE = (15, 15, 15)

HIT_FLASH = (255, 255, 255)

MAX_LIVES = 3
MAG_SIZE = 6
RELOAD_TIME_MS = 1200
SHOT_COOLDOWN_MS = 140

# difficulty progression
BASE_SPAWN_MS = 1050
MIN_SPAWN_MS = 350
BASE_DUCK_SPEED = 2.8
SPEED_PER_LEVEL = 0.35


# -----------------------------
# Helpers
# -----------------------------
def clamp(v, lo, hi):
    return max(lo, min(hi, v))


def draw_text(surface, text, pos, font, color=HUD_COLOR):
    img = font.render(text, True, color)
    surface.blit(img, pos)


def circle_hit(px, py, cx, cy, r):
    return (px - cx) ** 2 + (py - cy) ** 2 <= r ** 2


# -----------------------------
# Game Objects
# -----------------------------
@dataclass
class Duck:
    x: float
    y: float
    vx: float
    vy: float
    r: float
    wobble_phase: float
    worth: int
    alive: bool = True

    def update(self):
        if not self.alive:
            return
        self.x += self.vx
        self.y += self.vy

        # sine wobble to feel "ducky"
        self.wobble_phase += 0.08
        self.y += math.sin(self.wobble_phase) * 0.6

    def offscreen(self):
        # A bit generous so ducks "fully" leave
        return self.x < -120 or self.x > WIDTH + 120 or self.y < -120 or self.y > HEIGHT + 120

    def draw(self, surface):
        if not self.alive:
            return

        # body
        pygame.draw.circle(surface, DUCK_BODY, (int(self.x), int(self.y)), int(self.r))
        # wing
        wing_r = int(self.r * 0.72)
        pygame.draw.circle(surface, DUCK_WING, (int(self.x - self.r * 0.25), int(self.y + self.r * 0.10)), wing_r)
        # head
        head_r = int(self.r * 0.55)
        pygame.draw.circle(surface, DUCK_BODY, (int(self.x + self.r * 0.75), int(self.y - self.r * 0.15)), head_r)
        # beak
        beak = [
            (int(self.x + self.r * 1.35), int(self.y - self.r * 0.20)),
            (int(self.x + self.r * 1.85), int(self.y - self.r * 0.05)),
            (int(self.x + self.r * 1.35), int(self.y + self.r * 0.10)),
        ]
        pygame.draw.polygon(surface, DUCK_BEAK, beak)
        # eye
        pygame.draw.circle(surface, DUCK_EYE, (int(self.x + self.r * 0.92), int(self.y - self.r * 0.30)), max(2, int(self.r * 0.10)))


@dataclass
class HitSpark:
    x: float
    y: float
    ttl: int = 10

    def update(self):
        self.ttl -= 1

    def draw(self, surface):
        # little expanding lines
        t = self.ttl
        size = 12 - t
        for ang in (0, 45, 90, 135):
            rad = math.radians(ang)
            dx = math.cos(rad) * size
            dy = math.sin(rad) * size
            pygame.draw.line(surface, SHOT_COLOR, (self.x - dx, self.y - dy), (self.x + dx, self.y + dy), 2)


# -----------------------------
# Main Game
# -----------------------------
class Game:
    def __init__(self):
        pygame.init()
        pygame.display.set_caption("Shoot the Duck")
        self.screen = pygame.display.set_mode((WIDTH, HEIGHT))
        self.clock = pygame.time.Clock()

        self.font = pygame.font.SysFont("consolas", 22)
        self.big_font = pygame.font.SysFont("consolas", 56)

        self.reset()

    def reset(self):
        self.score = 0
        self.lives = MAX_LIVES
        self.level = 1

        self.ducks: list[Duck] = []
        self.sparks: list[HitSpark] = []

        self.mag = MAG_SIZE
        self.reloading = False
        self.reload_end_time = 0
        self.last_shot_time = 0

        self.paused = False
        self.game_over = False

        self.next_spawn_time = pygame.time.get_ticks() + 800
        self.flash_timer = 0

    def spawn_duck(self):
        # spawn from left or right with random altitude
        from_left = random.random() < 0.5

        y = random.randint(70, int(HEIGHT * 0.65))
        x = -80 if from_left else WIDTH + 80

        speed = BASE_DUCK_SPEED + (self.level - 1) * SPEED_PER_LEVEL
        vx = speed if from_left else -speed
        vy = random.uniform(-0.8, 0.8)

        r = random.uniform(18, 28)

        # worth scales slightly with speed/size
        worth = int(10 + (speed * 3) + (28 - r))
        worth = clamp(worth, 10, 40)

        duck = Duck(x=x, y=y, vx=vx, vy=vy, r=r, wobble_phase=random.random() * 10, worth=worth)
        self.ducks.append(duck)

        # spawn pacing increases with level
        now = pygame.time.get_ticks()
        spawn_ms = max(MIN_SPAWN_MS, BASE_SPAWN_MS - (self.level - 1) * 70)
        jitter = random.randint(-120, 120)
        self.next_spawn_time = now + spawn_ms + jitter

    def begin_reload(self):
        if self.reloading:
            return
        self.reloading = True
        self.reload_end_time = pygame.time.get_ticks() + RELOAD_TIME_MS

    def finish_reload_if_ready(self):
        if self.reloading and pygame.time.get_ticks() >= self.reload_end_time:
            self.reloading = False
            self.mag = MAG_SIZE

    def handle_shot(self, mx, my):
        now = pygame.time.get_ticks()
        if self.reloading or self.game_over or self.paused:
            return
        if now - self.last_shot_time < SHOT_COOLDOWN_MS:
            return
        if self.mag <= 0:
            self.begin_reload()
            return

        self.last_shot_time = now
        self.mag -= 1
        self.sparks.append(HitSpark(mx, my))
        self.flash_timer = 3

        # find closest duck hit (so overlapping feels fair)
        hit_ducks = []
        for d in self.ducks:
            if d.alive and circle_hit(mx, my, d.x, d.y, d.r * 1.05):
                dist2 = (mx - d.x) ** 2 + (my - d.y) ** 2
                hit_ducks.append((dist2, d))

        if hit_ducks:
            hit_ducks.sort(key=lambda t: t[0])
            d = hit_ducks[0][1]
            d.alive = False
            self.score += d.worth

            # level up every N points (simple ramp)
            new_level = 1 + self.score // 250
            if new_level > self.level:
                self.level = new_level

        # auto reload after last shot in mag
        if self.mag == 0:
            self.begin_reload()

    def update(self):
        now = pygame.time.get_ticks()
        self.finish_reload_if_ready()

        if self.flash_timer > 0:
            self.flash_timer -= 1

        # spawn ducks
        if not self.paused and not self.game_over and now >= self.next_spawn_time:
            self.spawn_duck()

        # update ducks
        if not self.paused and not self.game_over:
            for d in self.ducks:
                d.update()

        # remove offscreen ducks and penalize if alive escaped
        kept = []
        for d in self.ducks:
            if d.offscreen():
                if d.alive:
                    self.lives -= 1
                continue
            kept.append(d)
        self.ducks = kept

        # sparks
        if not self.paused:
            for s in self.sparks:
                s.update()
        self.sparks = [s for s in self.sparks if s.ttl > 0]

        if self.lives <= 0:
            self.game_over = True

    def draw_background(self):
        self.screen.fill(BG_COLOR)
        # ground
        pygame.draw.rect(self.screen, GROUND_COLOR, (0, int(HEIGHT * 0.78), WIDTH, int(HEIGHT * 0.22)))

        # some clouds
        for i in range(4):
            cx = (i * 240 + 120 + (pygame.time.get_ticks() // 30) % 240) % (WIDTH + 200) - 100
            cy = 70 + i * 20
            pygame.draw.circle(self.screen, (245, 245, 245), (cx, cy), 26)
            pygame.draw.circle(self.screen, (245, 245, 245), (cx + 25, cy + 10), 22)
            pygame.draw.circle(self.screen, (245, 245, 245), (cx - 25, cy + 10), 22)

        if self.flash_timer > 0:
            flash = pygame.Surface((WIDTH, HEIGHT), pygame.SRCALPHA)
            flash.fill((255, 255, 255, 50))
            self.screen.blit(flash, (0, 0))

    def draw_hud(self):
        # top bar text
        draw_text(self.screen, f"Score: {self.score}", (16, 12), self.font)
        draw_text(self.screen, f"Level: {self.level}", (16, 40), self.font)
        draw_text(self.screen, f"Lives: {self.lives}", (16, 68), self.font)

        # ammo
        ammo_text = f"Ammo: {self.mag}/{MAG_SIZE}"
        if self.reloading:
            remaining = max(0, self.reload_end_time - pygame.time.get_ticks())
            ammo_text = f"Reloading... {remaining/1000:.1f}s"
        draw_text(self.screen, ammo_text, (WIDTH - 260, 12), self.font)

        draw_text(self.screen, "LMB: shoot  |  R: reload  |  P: pause  |  ESC: quit", (WIDTH - 585, 40), self.font)

        if self.paused and not self.game_over:
            msg = self.big_font.render("PAUSED", True, (0, 0, 0))
            self.screen.blit(msg, msg.get_rect(center=(WIDTH // 2, HEIGHT // 2)))

        if self.game_over:
            msg = self.big_font.render("GAME OVER", True, (0, 0, 0))
            self.screen.blit(msg, msg.get_rect(center=(WIDTH // 2, HEIGHT // 2 - 40)))
            sub = self.font.render("Press ENTER to restart", True, (0, 0, 0))
            self.screen.blit(sub, sub.get_rect(center=(WIDTH // 2, HEIGHT // 2 + 20)))

    def draw_crosshair(self, mx, my):
        # nice little crosshair
        r = 14
        pygame.draw.circle(self.screen, CROSSHAIR_COLOR, (mx, my), r, 2)
        pygame.draw.line(self.screen, CROSSHAIR_COLOR, (mx - 22, my), (mx - 6, my), 2)
        pygame.draw.line(self.screen, CROSSHAIR_COLOR, (mx + 6, my), (mx + 22, my), 2)
        pygame.draw.line(self.screen, CROSSHAIR_COLOR, (mx, my - 22), (mx, my - 6), 2)
        pygame.draw.line(self.screen, CROSSHAIR_COLOR, (mx, my + 6), (mx, my + 22), 2)

    def render(self):
        self.draw_background()

        for d in self.ducks:
            d.draw(self.screen)

        for s in self.sparks:
            s.draw(self.screen)

        self.draw_hud()

        mx, my = pygame.mouse.get_pos()
        self.draw_crosshair(mx, my)

        pygame.display.flip()

    def run(self):
        pygame.mouse.set_visible(False)

        while True:
            self.clock.tick(FPS)

            # events
            for event in pygame.event.get():
                if event.type == pygame.QUIT:
                    pygame.quit()
                    sys.exit()

                if event.type == pygame.KEYDOWN:
                    if event.key == pygame.K_ESCAPE:
                        pygame.quit()
                        sys.exit()
                    if event.key == pygame.K_p:
                        if not self.game_over:
                            self.paused = not self.paused
                    if event.key == pygame.K_r:
                        if not self.game_over and not self.paused:
                            self.begin_reload()
                    if event.key == pygame.K_RETURN:
                        if self.game_over:
                            self.reset()

                if event.type == pygame.MOUSEBUTTONDOWN and event.button == 1:
                    mx, my = pygame.mouse.get_pos()
                    self.handle_shot(mx, my)

            if not self.paused and not self.game_over:
                self.update()
            else:
                # still tick sparks down a bit if paused? (kept simple)
                self.finish_reload_if_ready()

            self.render()


if __name__ == "__main__":
    Game().run()
