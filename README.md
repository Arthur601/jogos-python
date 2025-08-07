#pingpong

import pygame
import sys

pygame.init()

WIDTH, HEIGHT = 800, 400
screen = pygame.display.set_mode((WIDTH, HEIGHT))
pygame.display.set_caption("Joguinho de Luta com Facas")

WHITE = (255, 255, 255)
RED = (220, 20, 60)
BLUE = (65, 105, 225)
BLACK = (0, 0, 0)
GREEN = (0, 255, 0)
YELLOW = (255, 215, 0)
PURPLE = (128, 0, 128)

clock = pygame.time.Clock()
FPS = 60

GROUND_Y = HEIGHT - 60

class Knife:
    def __init__(self, x, y, facing_right):
        self.width = 30
        self.height = 10
        self.color = (100, 100, 100)
        self.speed = 15
        self.facing_right = facing_right
        self.x = x
        self.y = y
        self.rect = pygame.Rect(self.x, self.y, self.width, self.height)
        self.active = True  # Se está na tela e pode causar dano

    def update(self):
        if self.facing_right:
            self.x += self.speed
        else:
            self.x -= self.speed
        self.rect.x = self.x
        # Desativa se sair da tela
        if self.x > WIDTH or self.x + self.width < 0:
            self.active = False

    def draw(self, surface):
        pygame.draw.rect(surface, self.color, self.rect)

class Fighter:
    def __init__(self, x, y, color, controls):
        self.x = x
        self.y = y
        self.width = 50
        self.height = 100
        self.color = color
        self.speed = 5
        self.vel_y = 0
        self.jump_power = 15
        self.gravity = 1
        self.is_jumping = False
        self.health = 100
        self.max_health = 100

        self.attack_cooldown = 0
        self.attack_box = None
        self.is_attacking = False

        self.controls = controls
        self.facing_right = True

        # Poder especial: faca
        self.special_ready = True
        self.special_cooldown = 0
        self.knives = []  # lista de facas lançadas (limitaremos 1 por vez)
        self.special_damage = 8

    def draw(self, surface):
        pygame.draw.rect(surface, self.color, (self.x, self.y, self.width, self.height))

        # Vida
        health_bar_width = 100
        health_ratio = self.health / self.max_health
        pygame.draw.rect(surface, RED, (self.x, self.y - 20, health_bar_width, 10))
        pygame.draw.rect(surface, GREEN, (self.x, self.y - 20, health_bar_width * health_ratio, 10))

        # Barra especial
        special_bar_width = 100
        if self.special_ready:
            pygame.draw.rect(surface, PURPLE, (self.x, self.y - 35, special_bar_width, 6))
        else:
            cooldown_ratio = 1 - self.special_cooldown / (FPS * 5)
            if cooldown_ratio < 0: cooldown_ratio = 0
            pygame.draw.rect(surface, (100, 0, 100), (self.x, self.y - 35, special_bar_width, 6))
            pygame.draw.rect(surface, PURPLE, (self.x, self.y - 35, special_bar_width * cooldown_ratio, 6))

        # Ataque corpo a corpo
        if self.is_attacking and self.attack_box:
            pygame.draw.rect(surface, YELLOW, self.attack_box)

        # Desenha facas
        for knife in self.knives:
            if knife.active:
                knife.draw(surface)

    def move(self, keys):
        if keys[self.controls['left']] and self.x > 0:
            self.x -= self.speed
            self.facing_right = False
        if keys[self.controls['right']] and self.x < WIDTH - self.width:
            self.x += self.speed
            self.facing_right = True
        if not self.is_jumping and keys[self.controls['jump']]:
            self.vel_y = -self.jump_power
            self.is_jumping = True

    def apply_gravity(self):
        self.y += self.vel_y
        self.vel_y += self.gravity
        if self.y >= GROUND_Y - self.height:
            self.y = GROUND_Y - self.height
            self.is_jumping = False
            self.vel_y = 0

    def attack(self):
        if self.attack_cooldown == 0:
            self.is_attacking = True
            self.attack_cooldown = 20
            attack_width = 30
            attack_height = 20
            if self.facing_right:
                attack_x = self.x + self.width
            else:
                attack_x = self.x - attack_width
            attack_y = self.y + self.height // 2 - attack_height // 2
            self.attack_box = pygame.Rect(attack_x, attack_y, attack_width, attack_height)

    def use_special(self):
        # Lança uma faca se puder (só 1 faca por vez)
        if self.special_ready and len(self.knives) == 0:
            knife_x = self.x + self.width if self.facing_right else self.x - 30
            knife_y = self.y + self.height // 2 - 5
            new_knife = Knife(knife_x, knife_y, self.facing_right)
            self.knives.append(new_knife)
            self.special_ready = False
            self.special_cooldown = FPS * 5  # 5 segundos para recarregar

    def cooldowns(self):
        if self.attack_cooldown > 0:
            self.attack_cooldown -= 1
        else:
            self.is_attacking = False
            self.attack_box = None

        if self.special_cooldown > 0:
            self.special_cooldown -= 1
        else:
            self.special_ready = True

        # Atualiza facas e remove inativas
        for knife in self.knives:
            knife.update()
        self.knives = [k for k in self.knives if k.active]

    def get_rect(self):
        return pygame.Rect(self.x, self.y, self.width, self.height)

controls1 = {'left': pygame.K_a, 'right': pygame.K_d, 'jump': pygame.K_w, 'attack': pygame.K_s, 'special': pygame.K_q}
controls2 = {'left': pygame.K_LEFT, 'right': pygame.K_RIGHT, 'jump': pygame.K_UP, 'attack': pygame.K_DOWN, 'special': pygame.K_RCTRL}

player1 = Fighter(100, GROUND_Y - 100, BLUE, controls1)
player2 = Fighter(600, GROUND_Y - 100, RED, controls2)

font = pygame.font.SysFont(None, 36)

def draw_window():
    screen.fill(WHITE)
    pygame.draw.rect(screen, BLACK, (0, GROUND_Y, WIDTH, HEIGHT - GROUND_Y))
    player1.draw(screen)
    player2.draw(screen)
    health_text1 = font.render(f"P1 HP: {player1.health}", True, BLACK)
    health_text2 = font.render(f"P2 HP: {player2.health}", True, BLACK)
    screen.blit(health_text1, (20, 20))
    screen.blit(health_text2, (WIDTH - 150, 20))
    pygame.display.update()

def check_attacks():
    # Ataques corpo a corpo
    if player1.is_attacking and player1.attack_box.colliderect(player2.get_rect()):
        player2.health -= 1
    if player2.is_attacking and player2.attack_box.colliderect(player1.get_rect()):
        player1.health -= 1

    # Facas especiais
    for knife in player1.knives:
        if knife.active and knife.rect.colliderect(player2.get_rect()):
            player2.health -= player1.special_damage
            knife.active = False  # faca some depois do dano
    for knife in player2.knives:
        if knife.active and knife.rect.colliderect(player1.get_rect()):
            player1.health -= player2.special_damage
            knife.active = False

def check_winner():
    if player1.health <= 0:
        return "Jogador 2 venceu!"
    elif player2.health <= 0:
        return "Jogador 1 venceu!"
    return None

def main():
    run = True
    winner_text = None

    while run:
        clock.tick(FPS)
        keys = pygame.key.get_pressed()

        for event in pygame.event.get():
            if event.type == pygame.QUIT:
                run = False
            if event.type == pygame
