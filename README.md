# FauxM0n.c
Gba rom
#include <gba.h>
#include <string.h>

#define SCREEN_W 240
#define SCREEN_H 160
#define VRAM ((u16*)0x6000000)

typedef enum {
    STATE_OVERWORLD,
    STATE_BATTLE
} GameState;

typedef struct {
    char name[12];
    int hp;
    int max_hp;
    int level;
    int attack;
    int defense;
} Monster;

static GameState state = STATE_OVERWORLD;
static int player_x = 112;
static int player_y = 75;
static int party_count = 1;
static int encounters = 0;
static int encounter_timer = 0;
static unsigned int seed = 42;

static Monster starter = {
    "Startermon", 25, 25, 5, 12, 8
};

static Monster wild;

static u16 color(int r, int g, int b) {
    return RGB15(r, g, b);
}

static unsigned int random_number(void) {
    seed = seed * 1103515245 + 12345;
    return (seed >> 16) & 32767;
}

static void rectangle(int x, int y, int w, int h, u16 c) {
    int xx, yy;

    for (yy = y; yy < y + h; yy++) {
        if (yy < 0 || yy >= SCREEN_H) continue;

        for (xx = x; xx < x + w; xx++) {
            if (xx >= 0 && xx < SCREEN_W) {
                VRAM[yy * SCREEN_W + xx] = c;
            }
        }
    }
}

static void clear_screen(u16 c) {
    int i;

    for (i = 0; i < SCREEN_W * SCREEN_H; i++) {
        VRAM[i] = c;
    }
}

static void draw_player(void) {
    u16 skin = color(31, 20, 12);
    u16 clothes = color(5, 10, 31);

    rectangle(player_x, player_y, 16, 19, skin);
    rectangle(player_x, player_y + 8, 16, 11, clothes);
    rectangle(player_x + 3, player_y + 2, 3, 3, color(0, 0, 0));
    rectangle(player_x + 10, player_y + 2, 3, 3, color(0, 0, 0));
}

static void draw_monster(int x, int y, u16 c) {
    rectangle(x + 5, y, 24, 8, c);
    rectangle(x, y + 7, 34, 22, c);
    rectangle(x + 5, y + 27, 8, 7, c);
    rectangle(x + 22, y + 27, 8, 7, c);

    rectangle(x + 7, y + 11, 4, 5, color(0, 0, 0));
    rectangle(x + 23, y + 11, 4, 5, color(0, 0, 0));
}

static void draw_overworld(void) {
    int x, y;

    clear_screen(color(5, 23, 7));

    // Grass pattern
    for (y = 0; y < 160; y += 16) {
        for (x = 0; x < 240; x += 16) {
            if (((x + y) / 16) % 2 == 0) {
                rectangle(x, y, 8, 8, color(7, 25, 8));
            }
        }
    }

    // Walking path
    rectangle(0, 125, 240, 35, color(22, 17, 9));

    // Grass patches
    rectangle(25, 30, 35, 18, color(8, 27, 9));
    rectangle(170, 42, 42, 20, color(8, 27, 9));

    draw_player();

    // Party indicator
    rectangle(5, 5, 45, 8, color(31, 31, 31));
    rectangle(8, 8, party_count * 5, 3, color(0, 15, 31));
}

static void draw_hp_bar(int x, int y, int hp, int max_hp) {
    int width = 55;

    rectangle(x, y, width, 7, color(0, 0, 0));
    rectangle(x + 1, y + 1, width - 2, 5, color(20, 3, 3));

    if (hp > 0 && max_hp > 0) {
        int fill = (hp * (width - 2)) / max_hp;
        rectangle(x + 1, y + 1, fill, 5, color(5, 29, 5));
    }
}

static void spawn_wild_monster(void) {
    strcpy(wild.name, "Fangling");
    wild.level = 3;
    wild.hp = 15;
    wild.max_hp = 15;
    wild.attack = 6;
    wild.defense = 4;
}

static void draw_battle(void) {
    clear_screen(color(20, 25, 31));

    // Battle platforms
    rectangle(0, 112, 240, 48, color(9, 22, 12));
    rectangle(15, 105, 90, 12, color(12, 27, 15));
    rectangle(135, 57, 90, 12, color(12, 27, 15));

    // Wild monster and player monster
    draw_monster(164, 72, color(27, 10, 8));
    draw_monster(45, 110, color(5, 15, 29));

    // Health bars
    rectangle(8, 8, 85, 26, color(31, 31, 31));
    draw_hp_bar(20, 23, wild.hp, wild.max_hp);

    rectangle(147, 104, 85, 26, color(31, 31, 31));
    draw_hp_bar(160, 119, starter.hp, starter.max_hp);

    // Battle controls
    rectangle(0, 135, 240, 25, color(5, 7, 18));
    rectangle(8, 140, 65, 14, color(3, 15, 31));
    rectangle(87, 140, 65, 14, color(2, 25, 8));
    rectangle(166, 140, 65, 14, color(25, 15, 2));
}

int main(void) {
    irqInit();
    irqEnable(IRQ_VBLANK);

    REG_DISPCNT = MODE_3 | BG2_ENABLE;

    while (1) {
        VBlankIntrWait();

        scanKeys();

        u16 keys = keysDown();
        u16 held = keysHeld();

        if (state == STATE_OVERWORLD) {
            int moving = 0;

            if (held & KEY_LEFT && player_x > 2) {
                player_x -= 2;
                moving = 1;
            }

            if (held & KEY_RIGHT && player_x < 222) {
                player_x += 2;
                moving = 1;
            }

            if (held & KEY_UP && player_y > 2) {
                player_y -= 2;
                moving = 1;
            }

            if (held & KEY_DOWN && player_y < 136) {
                player_y += 2;
                moving = 1;
            }

            if (moving) {
                encounter_timer++;

                if (encounter_timer >= 15 &&
                    random_number() % 100 < 3) {
                    spawn_wild_monster();
                    state = STATE_BATTLE;
                    encounter_timer = 0;
                }
            }

            draw_overworld();
        }

        else if (state == STATE_BATTLE) {
            if (keys & KEY_A) {
                int damage = starter.attack - wild.defense / 2;

                if (damage < 1) damage = 1;

                wild.hp -= damage;

                if (wild.hp <= 0) {
                    wild.hp = 0;
                    encounters++;
                    state = STATE_OVERWORLD;
                } else {
                    int counter = wild.attack - starter.defense / 2;

                    if (counter < 1) counter = 1;

                    starter.hp -= counter;

                    if (starter.hp <= 0) {
                        starter.hp = starter.max_hp;
                        state = STATE_OVERWORLD;
                    }
                }
            }

            if (keys & KEY_B) {
                if (party_count < 6 && random_number() % 256 < 180) {
                    party_count++;
                    encounters++;
                    state = STATE_OVERWORLD;
                } else {
                    starter.hp -= 2;

                    if (starter.hp <= 0) {
                        starter.hp = starter.max_hp;
                        state = STATE_OVERWORLD;
                    }
                }
            }

            if (keys & KEY_START) {
                state = STATE_OVERWORLD;
            }

            if (state == STATE_BATTLE) {
                draw_battle();
            } else {
                draw_overworld();
            }
        }
    }

    return 0;
}
