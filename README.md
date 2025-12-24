# ============================================================
# ZOMBIE RUSH — JUEGO ARCADE COMPLETO EN PYTHON (PYGAME)
# Proyecto final — Código 100% funcional
# ============================================================

import pygame as pg
import random
import math
import sys

# ---------------- CONFIGURACIÓN GENERAL ----------------
WIDTH, HEIGHT = 960, 540
FPS = 60

BG_COLOR = (15, 15, 25)
PLAYER_COLOR = (70, 200, 255)
ZOMBIE_COLOR = (120, 255, 120)
FAST_ZOMBIE_COLOR = (255, 180, 80)
TANK_ZOMBIE_COLOR = (180, 180, 255)
BULLET_COLOR = (255, 255, 120)
CURE_COLOR = (255, 80, 80)
UI_COLOR = (240, 240, 240)

PLAYER_SPEED = 4
BULLET_SPEED = 11

pg.init()
screen = pg.display.set_mode((WIDTH, HEIGHT))
pg.display.set_caption("Zombie Rush")
clock = pg.time.Clock()

FONT = pg.font.SysFont("consolas", 18)
BIG_FONT = pg.font.SysFont("consolas", 44)

# ---------------- CLASE JUGADOR ----------------
class Player(pg.sprite.Sprite):
    def __init__(self):
        super().__init__()
        self.image = pg.Surface((40, 40), pg.SRCALPHA)
        pg.draw.rect(self.image, PLAYER_COLOR, (0, 0, 40, 40), border_radius=8)
        self.rect = self.image.get_rect(center=(WIDTH//2, HEIGHT//2))
        self.pos = pg.Vector2(self.rect.center)

        self.speed = PLAYER_SPEED
        self.health = 100
        self.max_health = 100

        self.invulnerable = False
        self.invuln_timer = 0

        self.weapon = 1
        self.last_shot = 0
        self.shoot_cd = 220

        self.last_dash = 0
        self.dash_cd = 1200
        self.dash_dist = 120

    def update(self, keys):
        vel = pg.Vector2(0, 0)
        if keys[pg.K_w]: vel.y -= 1
        if keys[pg.K_s]: vel.y += 1
        if keys[pg.K_a]: vel.x -= 1
        if keys[pg.K_d]: vel.x += 1

        if vel.length_squared() > 0:
            vel = vel.normalize() * self.speed

        if (keys[pg.K_LSHIFT] or keys[pg.K_SPACE]) and self.can_dash():
            self.dash(vel)

        self.pos += vel
        self.pos.x = max(20, min(WIDTH-20, self.pos.x))
        self.pos.y = max(20, min(HEIGHT-20, self.pos.y))
        self.rect.center = self.pos

        if self.invuln_timer > 0:
            self.invuln_timer -= 1
            if self.invuln_timer == 0:
                self.invulnerable = False

    def can_dash(self):
        return pg.time.get_ticks() - self.last_dash > self.dash_cd

    def dash(self, vel):
        if vel.length_squared() == 0:
            return
        self.last_dash = pg.time.get_ticks()
        self.pos += vel.normalize() * self.dash_dist

    def can_shoot(self):
        return pg.time.get_ticks() - self.last_shot > self.shoot_cd

    def shoot(self, target):
        if not self.can_shoot():
            return []

        self.last_shot = pg.time.get_ticks()
        direction = (pg.Vector2(target) - self.pos).normalize()
        bullets = []

        if self.weapon == 1:
            bullets.append(Bullet(self.pos, direction))

        elif self.weapon == 2:
            for ang in [-12, 0, 12]:
                bullets.append(Bullet(self.pos, direction.rotate(ang)))

        elif self.weapon == 3:
            for ang in [-20, -10, 0, 10, 20]:
                bullets.append(Bullet(self.pos, direction.rotate(ang)))

        return bullets

# ---------------- BALAS ----------------
class Bullet(pg.sprite.Sprite):
    def __init__(self, pos, direction):
        super().__init__()
        self.image = pg.Surface((10, 4), pg.SRCALPHA)
        pg.draw.rect(self.image, BULLET_COLOR, (0, 0, 10, 4), border_radius=2)
        angle = math.degrees(math.atan2(-direction.y, direction.x))
        self.image = pg.transform.rotate(self.image, angle)
        self.rect = self.image.get_rect(center=pos)
        self.pos = pg.Vector2(pos)
        self.dir = direction
        self.speed = BULLET_SPEED

    def update(self):
        self.pos += self.dir * self.speed
        self.rect.center = self.pos
        if not screen.get_rect().collidepoint(self.pos):
            self.kill()

# ---------------- ZOMBIS ----------------
class Zombie(pg.sprite.Sprite):
    def __init__(self, speed, hp, color):
        super().__init__()
        self.image = pg.Surface((36, 36), pg.SRCALPHA)
        pg.draw.rect(self.image, color, (0, 0, 36, 36), border_radius=6)
        side = random.choice(["l","r","t","b"])
        if side == "l": self.pos = pg.Vector2(-40, random.randint(0, HEIGHT))
        if side == "r": self.pos = pg.Vector2(WIDTH+40, random.randint(0, HEIGHT))
        if side == "t": self.pos = pg.Vector2(random.randint(0, WIDTH), -40)
        if side == "b": self.pos = pg.Vector2(random.randint(0, WIDTH), HEIGHT+40)
        self.rect = self.image.get_rect(center=self.pos)
        self.speed = speed
        self.hp = hp

    def update(self, player):
        direction = (player.pos - self.pos).normalize()
        self.pos += direction * self.speed
        self.rect.center = self.pos

# ---------------- CURACIÓN ----------------
class Cure(pg.sprite.Sprite):
    def __init__(self, pos):
        super().__init__()
        self.image = pg.Surface((22, 22), pg.SRCALPHA)
        pg.draw.rect(self.image, CURE_COLOR, (0, 0, 22, 22), border_radius=5)
        self.rect = self.image.get_rect(center=pos)

# ---------------- JUEGO PRINCIPAL ----------------
player = Player()
all_sprites = pg.sprite.Group(player)
bullets = pg.sprite.Group()
zombies = pg.sprite.Group()
cures = pg.sprite.Group()

score = 0
wave = 1
spawn_timer = 0
game_over = False
paused = False

def spawn_wave(w):
    for _ in range(w * 2):
        ztype = random.choice(["normal", "fast", "tank"])
        if ztype == "normal":
            zombies.add(Zombie(1.6, 30, ZOMBIE_COLOR))
        elif ztype == "fast":
            zombies.add(Zombie(2.6, 20, FAST_ZOMBIE_COLOR))
        else:
            zombies.add(Zombie(1.0, 80, TANK_ZOMBIE_COLOR))
        all_sprites.add(zombies)

# ---------------- LOOP PRINCIPAL ----------------
running = True
while running:
    clock.tick(FPS)
    for event in pg.event.get():
        if event.type == pg.QUIT:
            running = False

        if event.type == pg.KEYDOWN:
            if event.key == pg.K_1: player.weapon = 1
            if event.key == pg.K_2: player.weapon = 2
            if event.key == pg.K_3: player.weapon = 3
            if event.key == pg.K_p: paused = not paused
            if event.key == pg.K_r and game_over:
                all_sprites.empty()
                bullets.empty()
                zombies.empty()
                cures.empty()
                player = Player()
                all_sprites.add(player)
                score = 0
                wave = 1
                game_over = False

        if event.type == pg.MOUSEBUTTONDOWN and not game_over:
            for b in player.shoot(pg.mouse.get_pos()):
                bullets.add(b)
                all_sprites.add(b)

    if paused or game_over:
        screen.fill(BG_COLOR)
        text = BIG_FONT.render("PAUSA" if paused else "GAME OVER", True, UI_COLOR)
        screen.blit(text, text.get_rect(center=(WIDTH//2, HEIGHT//2)))
        pg.display.flip()
        continue

    keys = pg.key.get_pressed()
    player.update(keys)

    if not zombies:
        wave += 1
        spawn_wave(wave)

    for z in zombies:
        z.update(player)

    bullets.update()

    for z in pg.sprite.groupcollide(zombies, bullets, False, True):
        z.hp -= 30
        if z.hp <= 0:
            score += 10
            if random.random() < 0.2:
                c = Cure(z.pos)
                cures.add(c)
                all_sprites.add(c)
            z.kill()

    if not player.invulnerable:
        hits = pg.sprite.spritecollide(player, zombies, False)
        if hits:
            player.health -= 10
            player.invulnerable = True
            player.invuln_timer = 60
            if player.health <= 0:
                game_over = True

    for c in pg.sprite.spritecollide(player, cures, True):
        player.health = min(100, player.health + 15)

    screen.fill(BG_COLOR)
    all_sprites.draw(screen)

    # UI
    pg.draw.rect(screen, (80,80,80), (20,20,200,18))
    pg.draw.rect(screen, (120,255,120), (20,20,200*(player.health/100),18))
    screen.blit(FONT.render(f"HP: {player.health}", True, UI_COLOR), (20,2))
    screen.blit(FONT.render(f"Score: {score}", True, UI_COLOR), (20,45))
    screen.blit(FONT.render(f"Wave: {wave}", True, UI_COLOR), (20,65))
    screen.blit(FONT.render(f"Weapon: {player.weapon}", True, UI_COLOR), (20,85))

    pg.display.flip()

pg.quit()
sys.exit()

