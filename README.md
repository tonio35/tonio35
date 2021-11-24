import pygame
import math
import sys
import time

from math import *

pygame.init()

#definit fenetre de jeu

width = 1800
height = 1000
display = pygame.display.set_mode((width, height))
pygame.display.set_caption("Pool Breaker")

# Definit le level 1  :  E>mur B,C,D>tuiles A>aimant M>mine X>Spawn Balle O>Gros rond et S>Sortie

level1 = [
        "O EEEEEEEEEEEEE O",
        "        A        ",
        "E   BCD     BCD E",
        "EBBCCDD     BCD E",
        "EBBCCDD         E",
        "EB MCDD  BMCCDDDE",
        "E       EBBCCMDDE",
        "EX AAAAAEBBCCDDSE",
        "EEEEEEEEEEEEEEEEE", ]

clock = pygame.time.Clock()
# Couleurs
white = (236, 240, 241)
black = (23, 32, 42)
red = (203, 67, 53)
gizmoColor = (230, 126, 34)
bleu = (90, 255, 255)
# Parametres du jeu
rayon_balle = 25
friction_balle = 0.04
vitesse_balle_max = 200
TILE_SIZE = 100   #Ne pas toucher a celui la
chrono_mine = 3
coup_restant = 20
largeur_gizmo = 6


# Fonts du jeu
arial_font = pygame.font.SysFont("arial Black", 36)
berlin_font = pygame.font.SysFont("Berlin Sans FB demi", 50)

#defnit les groupes de sprites

groupe_sprites_explosion = pygame.sprite.Group()
groupe_aimants = pygame.sprite.Group()
groupe_mines = pygame.sprite.Group()
groupe_tuiles = pygame.sprite.Group()
groupe_murs = pygame.sprite.Group()
groupe_sortie = pygame.sprite.Group()
groupe_rond = pygame.sprite.Group()
spawn_balle = []
position_sortie = []

"""             FONCTIONS           """

# creer le niveau de jeu en incluant les differents sprites dans leurs groupes respectifs
def create_level():

    x = y = TILE_SIZE

    for rangees in level1:
        for colonnes in rangees:
            if colonnes == "A":
                nouveau_aimant = Sprite('aimant.png', x, y)
                groupe_aimants.add(nouveau_aimant)
            if colonnes == "B":
                nouvelle_tuile = Sprite('tuile_verte.png', x, y)
                groupe_tuiles.add(nouvelle_tuile)
            if colonnes == "C":
                nouvelle_tuile = Sprite('tuile_orange.png', x, y)
                groupe_tuiles.add(nouvelle_tuile)
            if colonnes == "D":
                nouvelle_tuile = Sprite('tuile_rouge.png', x, y)
                groupe_tuiles.add(nouvelle_tuile)
            if colonnes == "E":
                nouveau_mur = Sprite("mur.png", x, y)
                groupe_murs.add(nouveau_mur)
            if colonnes == "M":
                nouvelle_mine = Sprite('mine4.png', x, y)
                groupe_mines.add(nouvelle_mine)
            if colonnes == "S":
                sortie = Sprite('trou.png', x, y)
                groupe_sortie.add(sortie)
                position_sortie.append(x)
                position_sortie.append(y)
            if colonnes == "O":
                sortie = Sprite('gros_rond.png', x, y)
                groupe_rond.add(sortie)
            if colonnes == "X":
                spawn_balle.append(x)
                spawn_balle.append(y)

            x += TILE_SIZE

        y += TILE_SIZE
        x = TILE_SIZE

# fonction game over
def game_over():
    text_game_over = berlin_font.render("PERDU", True, white)
    display.blit(text_game_over, [width/2-100, height/2])
    pygame.display.update()
    time.sleep(5)
    pygame.quit()
    sys.exit()
def level_up():
    text_game_over = berlin_font.render("GAGNE", True, white)
    display.blit(text_game_over, [width/2-100, height/2])
    pygame.display.update()
    time.sleep(3)
    pygame.quit()
    sys.exit()

# fonction afficher les textes du jeu
def textes_jeu():
    #definit et affiche les textes animes
    text = arial_font.render(f"FPS {int(clock.get_fps())}", True, black)
    text2 = arial_font.render(f"SCORE {balle.score}", True, black)
    text_sortie = arial_font.render(f"{coup_restant}", True, white)
    display.blit(text, [20, 50])
    display.blit(text2, [20, 100])
    display.blit(text_sortie, [position_sortie[0]-12, position_sortie[1]-27])

"""            CLASSES            """

# Gere la classe sprite du jeu
class Sprite(pygame.sprite.Sprite):
    def __init__(self, chemin_image, pos_x, pos_y):
        super().__init__()
        self.image = pygame.image.load(chemin_image)
        self.rect = self.image.get_rect()
        self.rect.center = [pos_x, pos_y]

# Gere l'explosion des mines en animant une suite de sprites
class Explosion(pygame.sprite.Sprite):
    def __init__(self, x, y):
        super().__init__()
        self.images = []
        for num in range (0, 43):
            img = pygame.image.load(f"explosion/explosion{num}.tga")
            self.images.append(img)
        self.index = 0
        self.image = self.images[self.index]
        self.rect = self.image.get_rect()
        self.rect.center = [x, y]
        self.counter = 0
    def update(self):
        explosion_speed = 1
        self.counter += 1
        if self.counter >= explosion_speed and self.index < len(self.images) - 1:
            self.counter = 0
            self.index += 1
            self.image = self.images[self.index]
        if self.index >= len(self.images) - 1 and self.counter >= explosion_speed:
            self.kill()
            if balle.explo_balle == True: game_over()

# Gere la bibliotheque de sons
class SoundManager:
    def __init__(self):
        self.sounds = {
            'rebond_tuile': pygame.mixer.Sound("sons/rebond_tuile.ogg"),
            'explosion': pygame.mixer.Sound("sons/explosion.ogg"),
            'gong': pygame.mixer.Sound("sons/gong.ogg"),
            'minuteur': pygame.mixer.Sound("sons/minuteur.ogg"),
            'rebond_mine': pygame.mixer.Sound("sons/rebond_mine.ogg"),
            'rebond_mur': pygame.mixer.Sound("sons/rebond_mur.ogg"),
            'aimant': pygame.mixer.Sound("sons/aimant.ogg"),
            'lance_balle': pygame.mixer.Sound("sons/lance_balle.ogg"),
            'chute': pygame.mixer.Sound("sons/chute.ogg"),


        }

    def player(self, name):
        self.sounds[name].play()

# Gere le repere (gizmo) angle et force pour le joueur
class Gizmo:
    def __init__(self, x, y, length, color):
        self.x = x
        self.y = y
        self.length = length
        self.color = color
        self.tangent = 0
        self.image = pygame.Surface([self.x, self.y])

    # fonction aplliquer force et angle choisi

    def applyForce(self, balle, force):
        balle.angle = self.tangent
        balle.speed = force


    # fonction dessiner le repere joueur (force et angle)

    def draw(self, cuex, cuey):


        self.x, self.y = pygame.mouse.get_pos()     #retour position souris

        self.tangent = int(degrees(atan2((self.y - cuey), (self.x - cuex))))      #calcule de la tangente entre la position souris et le centre de la balle

        self.longueur_repere = int(math.sqrt((self.x - cuex)**2 + (self.y - cuey)**2))       #calcule la distance entre la balle et souris

        # dessinne une ligne de repere pour que le joueur choisisse un angle et une force avec l'outil Line de pygame

        if self.longueur_repere < rayon_balle:
            pygame.draw.line(display, red, (cuex + int((rayon_balle + self.longueur_repere) * cos(radians(self.tangent))), cuey + int((rayon_balle + self.longueur_repere) * sin(radians(self.tangent)))), ((cuex + rayon_balle * cos(radians(self.tangent)), cuey + rayon_balle * sin(radians(self.tangent)))), largeur_gizmo)

        elif self.longueur_repere <= vitesse_balle_max :
            pygame.draw.line(display, gizmoColor, (cuex + int(self.longueur_repere * cos(radians(self.tangent))), cuey + int(self.longueur_repere * sin(radians(self.tangent)))), ((cuex + int(rayon_balle * cos(radians(self.tangent))), cuey + int(rayon_balle * sin(radians(self.tangent))))), largeur_gizmo)
        else:
            pygame.draw.line(display, gizmoColor, (cuex + int(vitesse_balle_max * cos(radians(self.tangent))), cuey + int(vitesse_balle_max * sin(radians(self.tangent)))), ((cuex + int(rayon_balle * cos(radians(self.tangent))), cuey + int(rayon_balle * sin(radians(self.tangent))))), largeur_gizmo)

# Gere la balle et ses collisions
class Balle:
    #definit le parametres initiaux de la balle
    def __init__(self, x, y, speed, color, angle):
        self.x = x
        self.y = y
        self.color = color
        self.angle = angle
        self.explo = False
        self.speed = speed
        self.score = 0
        self.explo_balle = False


    def move(self):

        #ralentit la balle en fonction de la friction

        self.speed -= friction_balle

        # arrete la balle au cas ou la vitesse soit negative a cause de la friction

        if self.speed <= 0:
            self.speed = 0

        # definit la trajectoire de la balle en fonction de son angle (sinus,cosinus) et de sa vitesse

        self.x = self.x + self.speed*cos(radians(self.angle))
        self.y = self.y + self.speed*sin(radians(self.angle))

        # Dessine la balle

        self.cercle = pygame.draw.circle(display, white, (self.x, self.y), rayon_balle)

        # Detection de collision entre la balle et les aimants avec la distance dans le groupe aimants

        for index_aimant in groupe_aimants:
            #calcul la distance entre la balle et l'aimant
            self.distance_balle_aimant = ((self.x - index_aimant.rect.center[0]) ** 2 + (self.y - index_aimant.rect.center[1]) ** 2) ** 0.5
            #detecte la colision entre la balle et l'aimant
            if self.distance_balle_aimant <= TILE_SIZE/2 + rayon_balle:

                soundmanager.player('aimant')
                #recul et stop la balle sur sa position d'avant la colision
                self.x = self.x - self.speed * cos(radians(self.angle))
                self.y = self.y - self.speed * sin(radians(self.angle))
                self.speed = 0

        # Detection de collision entre la balle et les gros ronds avec la distance

        for index_rond in groupe_rond:

            self.distance_balle_rond = ((self.x - index_rond.rect.center[0]) ** 2 + (self.y - index_rond.rect.center[1]) ** 2) ** 0.5

            if self.distance_balle_rond <= 150 + rayon_balle:
                soundmanager.player('gong')
                self.tangent_rond_balle = int(degrees((atan((index_rond.rect.center[1] - self.y) / (index_rond.rect.center[0] - self.x)))) + 90)
                self.angle = int(2 * self.tangent_rond_balle - self.angle)
                self.anglepos = int(self.tangent_rond_balle + 90)
                self.x -= (self.speed) * sin(radians(self.anglepos))
                self.y -= (self.speed) * cos(radians(self.anglepos))

        # Detection de collision entre la balle et les mines avec la distance dans le groupe mines

        for index_mine in groupe_mines:

            self.distance_balle_mine = ((self.x - index_mine.rect.center[0]) ** 2 + (self.y - index_mine.rect.center[1]) ** 2) ** 0.5

            if self.distance_balle_mine <= 50 + rayon_balle:
                soundmanager.player('rebond_mine')
                if chrono_mine == 3:soundmanager.player('minuteur')
                self.tangent_mine_balle = int(degrees((atan((index_mine.rect.center[1] - self.y)/(index_mine.rect.center[0] - self.x)))) + 90)
                self.angle = int(2 * self.tangent_mine_balle - self.angle)
                self.anglepos = int(self.tangent_mine_balle + 90)
                self.x -= (self.speed) * sin(radians(self.anglepos))
                self.y += (self.speed) * cos(radians(self.anglepos))

                groupe_mines.remove(index_mine)
                pygame.time.set_timer(pygame.USEREVENT, 1000)
                self.mine_chrono = index_mine
                self.mine_chrono_posx = index_mine.rect.center[0]
                self.mine_chrono_posy = index_mine.rect.center[1]
                self.explo = True

        if chrono_mine == 3 and self.explo == True:
            groupe_mines.remove(self.mine_chrono)
            nouvelle_mine = Sprite('mine3.png', self.mine_chrono_posx, self.mine_chrono_posy)
            groupe_mines.add(nouvelle_mine)
            self.mine_chrono = nouvelle_mine
        if chrono_mine == 2 :
            groupe_mines.remove(self.mine_chrono)
            nouvelle_mine = Sprite('mine2.png', self.mine_chrono_posx, self.mine_chrono_posy)
            groupe_mines.add(nouvelle_mine)
            self.mine_chrono = nouvelle_mine
        if chrono_mine == 1 :
            groupe_mines.remove(self.mine_chrono)
            nouvelle_mine = Sprite('mine1.png', self.mine_chrono_posx, self.mine_chrono_posy)
            groupe_mines.add(nouvelle_mine)
            self.mine_chrono = nouvelle_mine
        if chrono_mine == 0 and self.explo == True:
            soundmanager.player('explosion')
            groupe_mines.remove(self.mine_chrono)
            explosion = Explosion(self.mine_chrono_posx, self.mine_chrono_posy)
            groupe_sprites_explosion.add(explosion)
            self.explo = False
            #destruction des tuiles proches de la mine avec la distance
            for index_tuile in groupe_tuiles:
                self.distance_tuile_mine = ((index_tuile.rect.center[0] - self.mine_chrono_posx) ** 2 + (index_tuile.rect.center[1] - self.mine_chrono_posy) ** 2) ** 0.5

                if int(self.distance_tuile_mine) <= 250:
                    groupe_tuiles.remove(index_tuile)
            #detection si la balle est dans le rayon de l'explosion avec la distance
            self.distance_balle_mine = ((self.x - self.mine_chrono_posx) ** 2 + (self.y - self.mine_chrono_posy) ** 2) ** 0.5
            if  int(self.distance_balle_mine) < 250:self.explo_balle = True

        for index_sortie in groupe_sortie:
            self.distance_balle_sortie = ((self.x - index_sortie.rect.center[0]) ** 2 + (self.y - index_sortie.rect.center[1]) ** 2) ** 0.5
            if 25 < int(self.distance_balle_sortie) < 50 :soundmanager.player('chute')
            if self.distance_balle_sortie < 50:
                self.tangent_sortie_balle = degrees((atan((index_sortie.rect.center[1] - self.y) / ((index_sortie.rect.center[0] - self.x)+0.1))))
                self.angle = (2 * self.tangent_sortie_balle - self.angle)
                self.x = index_sortie.rect.center[0]
                self.y = index_sortie.rect.center[1]

        # Detection de collision entre la balle et les murs avec la position xy dans le groupe de sprites des murs

        for index_mur in groupe_murs:

            if self.y + rayon_balle > index_mur.rect.center[1] - TILE_SIZE/2 and self.y - rayon_balle < index_mur.rect.center[1] + TILE_SIZE/2 and self.x + rayon_balle > index_mur.rect.center[0] - TILE_SIZE/2 and self.x - rayon_balle < index_mur.rect.center[0] + TILE_SIZE/2 :

                soundmanager.player('rebond_mur')

                self.x = self.x - (self.speed*2) * cos(radians(self.angle))
                self.y = self.y - (self.speed*2) * sin(radians(self.angle))
                if self.y < index_mur.rect.center[1] - TILE_SIZE/2 or self.y > index_mur.rect.center[1] + TILE_SIZE/2 :
                    self.angle = 360 - int(self.angle)

                    return
                elif self.x < index_mur.rect.center[0] - TILE_SIZE/2 or self.x > index_mur.rect.center[0] + TILE_SIZE/2:
                    self.angle = 180 - int(self.angle)

                    return

        # Detection de collision entre la balle et les tuiles avec la position xy dans la groupe de sprites des tuiles

        for index_tuile in groupe_tuiles:
            #self.distance_balle_tuile = ((self.x - index_tuile.rect.center[0]) ** 2 + (self.y - index_tuile.rect.center[1]) ** 2) ** 0.5

            if self.y + rayon_balle > index_tuile.rect.center[1] - TILE_SIZE/2 and self.y - rayon_balle < index_tuile.rect.center[1] + TILE_SIZE/2 and self.x + rayon_balle > index_tuile.rect.center[0] - TILE_SIZE/2 and self.x - rayon_balle < index_tuile.rect.center[0] + TILE_SIZE/2 :
                groupe_tuiles.remove(index_tuile)
                self.score += 10
                soundmanager.player('rebond_tuile')
                self.x = self.x - self.speed * cos(radians(self.angle))
                self.y = self.y - self.speed * sin(radians(self.angle))

                if self.y < index_tuile.rect.center[1] - TILE_SIZE/2 or self.y > index_tuile.rect.center[1] + TILE_SIZE/2 :
                    self.angle = 360 - int(self.angle)

                    return

                elif self.x < index_tuile.rect.center[0] - TILE_SIZE/2 or self.x > index_tuile.rect.center[0] + TILE_SIZE/2 :
                    self.angle = 180 - int(self.angle)

                    return

"""               JEU             """

#Initialisation du jeu en creant le level et en placant la balle et le gizmo repere

create_level()
balle = Balle(spawn_balle[0], spawn_balle[1], 0, white, 0)
gizmo = Gizmo(0, 0, vitesse_balle_max, gizmoColor)
soundmanager = SoundManager()

# Boucle du jeu

while True:

    for event in pygame.event.get():
        if event.type == pygame.QUIT:
            pygame.quit()
            sys.exit()

        if event.type == pygame.MOUSEBUTTONDOWN and balle.speed == 0:
            soundmanager.player('lance_balle')
            coup_restant -= 1
            start = [balle.x, balle.y]
            x, y = pygame.mouse.get_pos()
            end = [x, y]
            dist = ((start[0] - end[0])**2 + (start[1] - end[1])**2)**0.5
            force = dist/10.0

            if force > vitesse_balle_max/10:
                force = vitesse_balle_max/10

            gizmo.applyForce(balle, force)

        if event.type == pygame.USEREVENT and chrono_mine >= 0:
            chrono_mine -= 1
            if chrono_mine == -1:
                pygame.time.set_timer(pygame.USEREVENT, 0, True)
                chrono_mine = 3

    # dessine les elements graphiques, le niveau de calque (buffered) est dans l'ordre d'affichage
    display.fill(black)
    groupe_aimants.draw(display)
    groupe_mines.draw(display)
    groupe_murs.draw(display)
    groupe_tuiles.draw(display)
    groupe_sortie.draw(display)
    groupe_rond.draw(display)
    balle.move()
    groupe_sprites_explosion.draw(display)
    groupe_sprites_explosion.update()
    textes_jeu()

    if not balle.speed > 0 :
        if coup_restant == 0 : game_over()
        if int(balle.x) == position_sortie[0] and int(balle.y) == position_sortie[1]: level_up()
        gizmo.draw(balle.x, balle.y)   #dessine le trait blanc (+texte) si vitesse =0

    #reglage du frame rate en FPS
    clock.tick(60)
    pygame.display.update()




