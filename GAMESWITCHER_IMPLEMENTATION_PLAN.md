# muOS GameSwitcher Implementation Plan

## Executive Summary

This document outlines a **test-driven development (TDD)** approach to port the OnionUI gameSwitcher feature to muOS. The gameSwitcher enables users to quickly switch between recently played games, manage save states, and resume gameplay without returning to the main menu—all from an overlay that can be triggered during active gameplay.

**Estimated Timeline**: 8-12 weeks (1 developer, full-time)  
**Primary Challenge**: UI framework migration (SDL 1.2 → LVGL)  
**Success Criteria**: Feature parity with OnionUI gameSwitcher + muOS integration tests passing

---

## Table of Contents

1. [Architecture Comparison](#architecture-comparison)
2. [Component Mapping](#component-mapping)
3. [Test-Driven Development Strategy](#test-driven-development-strategy)
4. [Implementation Phases](#implementation-phases)
5. [Technical Specifications](#technical-specifications)
6. [Integration Points](#integration-points)
7. [Testing Strategy](#testing-strategy)
8. [Risk Mitigation](#risk-mitigation)

---

## Architecture Comparison

### OnionUI GameSwitcher Stack

```
┌─────────────────────────────────────┐
│      gameSwitcher.c (Main Loop)     │
├─────────────────────────────────────┤
│  gs_history  │  gs_keystate  │      │
│  gs_render   │  gs_overlay   │      │
│  gs_popMenu  │  gs_retroarch │      │
├─────────────────────────────────────┤
│         SDL 1.2 Graphics            │
│  (Blitting, TTF, Image, Rotozoom)   │
├─────────────────────────────────────┤
│    Onion Common Libraries           │
│  (state, battery, theme, config)    │
├─────────────────────────────────────┤
│    Linux Framebuffer (/dev/fb0)     │
└─────────────────────────────────────┘
```

### muOS Equivalent Architecture

```
┌─────────────────────────────────────┐
│    muxswitcher.c (Main Module)      │
├─────────────────────────────────────┤
│  switcher_history │ switcher_input  │
│  switcher_ui      │ switcher_state  │
│  switcher_ra      │ switcher_saves  │
├─────────────────────────────────────┤
│         LVGL Graphics Framework     │
│  (Widgets, Styles, Animations)      │
├─────────────────────────────────────┤
│      muOS Common Libraries          │
│  (common, theme, config, device)    │
├─────────────────────────────────────┤
│    LVGL Display Driver (SDL2)       │
└─────────────────────────────────────┘
```

---

## Component Mapping

### File Structure Mapping

| **OnionUI File**       | **muOS Equivalent**              | **Purpose**                              |
|------------------------|----------------------------------|------------------------------------------|
| `gs_model.h`           | `switcher_model.h`               | Data structures (Game_s, RecentItem)     |
| `gs_history.h`         | `switcher_history.h/c`           | History parsing, game enumeration        |
| `gs_render.h`          | `switcher_ui.h/c`                | LVGL UI components                       |
| `gs_keystate.h`        | `switcher_input.h/c`             | Input handling, key combos               |
| `gs_overlay.h`         | `switcher_state.h/c`             | Overlay mode, pause/resume               |
| `gs_retroarch.h`       | `switcher_ra.h/c`                | RetroArch integration                    |
| `gs_popMenu.h`         | `switcher_saves.h/c`             | Save state management                    |
| `gs_romscreen.h`       | `switcher_screenshots.h/c`       | Screenshot caching, loading              |
| `gs_appState.h`        | Embedded in `muxswitcher.c`      | Application state management             |
| `gameSwitcher.c`       | `muxswitcher.c`                  | Main entry point, event loop             |

### Directory Structure

```
frontend/
├── module/
│   ├── muxswitcher.c                    # Main module (200-300 lines)
│   ├── switcher/
│   │   ├── switcher_model.h             # Data structures
│   │   ├── switcher_history.c/h         # History management
│   │   ├── switcher_ui.c/h              # LVGL UI components
│   │   ├── switcher_input.c/h           # Input handling
│   │   ├── switcher_state.c/h           # Overlay/state management
│   │   ├── switcher_ra.c/h              # RetroArch integration
│   │   ├── switcher_saves.c/h           # Save state menu
│   │   └── switcher_screenshots.c/h     # Screenshot system
│   └── ui/
│       ├── ui_muxswitcher.c/h           # LVGL UI definitions
│       └── assets/
│           └── switcher/                # Images, icons
├── common/
│   └── (existing shared libraries)
└── test/
    └── switcher/
        ├── test_history.c               # Unit tests
        ├── test_input.c
        ├── test_ra_integration.c
        └── test_ui_integration.c
```

---

## Test-Driven Development Strategy

### Testing Pyramid

```
        ┌──────────────┐
        │   E2E Tests  │  (5%)  Integration with muOS launcher
        ├──────────────┤
        │ UI Tests     │  (15%) LVGL widget behavior
        ├──────────────┤
        │ Integration  │  (30%) RetroArch, file I/O, threading
        ├──────────────┤
        │ Unit Tests   │  (50%) Core logic, data structures
        └──────────────┘
```

### Test Framework

**Tools:**
- **Unity** (C unit testing framework)
- **CMocka** (mocking framework)
- **LVGL Simulator** (UI testing on desktop)
- **valgrind** (memory leak detection)
- **gcov/lcov** (code coverage)

**Test Organization:**

```c
// test/switcher/test_history.c
void setUp(void) {
    // Create mock history file
}

void tearDown(void) {
    // Clean up temp files
}

void test_parseJsonToRecentItem_ValidEntry(void) {
    const char *json = "{\"type\":5,\"label\":\"Game\",\"rompath\":\"/path/rom.zip\"}";
    RecentItem item;
    
    bool result = parseJsonToRecentItem(json, &item, 1);
    
    TEST_ASSERT_TRUE(result);
    TEST_ASSERT_EQUAL_STRING("Game", item.label);
    TEST_ASSERT_EQUAL_INT(5, item.type);
}

void test_readHistory_DeduplicatesDuplicates(void) {
    // Create history with duplicates
    // Call readHistory()
    // Assert game_list_len == unique count
}
```

### TDD Workflow

**Red-Green-Refactor Cycle:**

```
1. RED:   Write failing test
2. GREEN: Write minimal code to pass test
3. REFACTOR: Improve code quality
4. REPEAT
```

**Example for History Parsing:**

```c
// Phase 1: Write test first (RED)
void test_readHistory_ParsesValidFile(void) {
    create_mock_history_file("test_history.lpl", 3);
    readHistory("test_history.lpl");
    TEST_ASSERT_EQUAL_INT(3, game_list_len);
}

// Phase 2: Implement minimal code (GREEN)
void readHistory(const char *path) {
    // Basic implementation to make test pass
}

// Phase 3: Refactor (REFACTOR)
void readHistory(const char *path) {
    // Add error handling, optimize, clean up
}
```

---

## Implementation Phases

### **Phase 1: Core Data & History (Weeks 1-2)**

#### Objectives
- Port data structures from OnionUI
- Implement history parsing with JSON support
- Create unit tests for core logic

#### Deliverables

1. **switcher_model.h** - Data Structures
```c
#ifndef MUXSWITCHER_MODEL_H
#define MUXSWITCHER_MODEL_H

#include "../common/common.h"
#include "../lvgl/lvgl.h"

#define MAX_HISTORY 100
#define PLAYTIME_DATA_PATH "/mnt/mmc/MUOS/info/tracker/playtime_data.json"  // muOS custom format
#define ROM_SCREENS_DIR "/mnt/mmc/MUOS/save/screenshots"  // TO BE VERIFIED ON HARDWARE
#define STATES_DIR "/mnt/mmc/MUOS/save/state"  // TO BE VERIFIED ON HARDWARE

typedef struct {
    char label[STR_MAX];
    char rompath[STR_MAX];
    char imgpath[STR_MAX];
    char launch[STR_MAX];
    int type;
    int lineNo;
} RecentItem;

typedef struct {
    RecentItem recent_item;
    lv_img_dsc_t *screenshot;     // LVGL image descriptor
    char rom_name[STR_MAX];
    char display_name[STR_MAX];
    char core_name[STR_MAX];
    char core_path[STR_MAX];
    char playtime[100];
    int index;
    bool processed;
    bool is_running;
} GameItem;

typedef struct {
    GameItem items[MAX_HISTORY];
    int count;
    int current_index;
    bool is_overlay;
    int view_mode;
} SwitcherState;

#endif
```

2. **switcher_history.c** - History Management
```c
#include "switcher_history.h"
#include "../common/json/json.h"

bool parseJsonToRecentItem(const char *json_str, RecentItem *item, int line_no) {
    cJSON *json = cJSON_Parse(json_str);
    if (!json) return false;
    
    // Parse JSON fields
    cJSON *type = cJSON_GetObjectItem(json, "type");
    if (!cJSON_IsNumber(type) || (type->valueint != 5 && type->valueint != 17)) {
        cJSON_Delete(json);
        return false;
    }
    
    // Extract fields...
    item->type = type->valueint;
    item->lineNo = line_no;
    
    cJSON_Delete(json);
    return true;
}

void readHistory(SwitcherState *state) {
    FILE *fp = fopen(HISTORY_PATH, "r");
    if (!fp) return;
    
    char line[MAX_BUFFER_SIZE];
    int line_no = 0;
    int count = 0;
    
    while (fgets(line, sizeof(line), fp) && count < MAX_HISTORY) {
        line_no++;
        
        if (!parseJsonToRecentItem(line, &state->items[count].recent_item, line_no))
            continue;
            
        // Check for duplicates
        bool is_dup = false;
        for (int i = 0; i < count; i++) {
            if (strcmp(state->items[i].recent_item.rompath,
                      state->items[count].recent_item.rompath) == 0) {
                is_dup = true;
                break;
            }
        }
        
        if (is_dup) continue;
        
        // Validate ROM exists
        if (!file_exist(state->items[count].recent_item.rompath))
            continue;
            
        state->items[count].index = count;
        state->items[count].processed = false;
        count++;
    }
    
    fclose(fp);
    state->count = count;
}
```

#### Tests

```c
// test/switcher/test_history.c

void test_parseJson_ValidEntry(void) {
    const char *json = "{\"type\":5,\"label\":\"Super Mario\",\"rompath\":\"/roms/mario.sfc\"}";
    RecentItem item;
    
    bool result = parseJsonToRecentItem(json, &item, 1);
    
    TEST_ASSERT_TRUE(result);
    TEST_ASSERT_EQUAL_STRING("Super Mario", item.label);
    TEST_ASSERT_EQUAL_INT(5, item.type);
}

void test_parseJson_InvalidType(void) {
    const char *json = "{\"type\":99,\"label\":\"Invalid\"}";
    RecentItem item;
    
    bool result = parseJsonToRecentItem(json, &item, 1);
    
    TEST_ASSERT_FALSE(result);
}

void test_readHistory_RemovesDuplicates(void) {
    // Create test file with duplicate entries
    create_test_history_file("test.lpl", (const char*[]){
        "{\"type\":5,\"rompath\":\"/rom1.zip\"}",
        "{\"type\":5,\"rompath\":\"/rom2.zip\"}",
        "{\"type\":5,\"rompath\":\"/rom1.zip\"}"  // Duplicate
    }, 3);
    
    SwitcherState state = {0};
    readHistory(&state);
    
    TEST_ASSERT_EQUAL_INT(2, state.count);
}
```

---

### **Phase 2: LVGL UI Components (Weeks 3-5)**

#### Objectives
- Convert SDL rendering to LVGL widgets
- Implement view modes (Normal, Minimal, Fullscreen)
- Create responsive layout system

#### Deliverables

1. **switcher_ui.h/c** - UI Components

**Header:**
```c
#ifndef MUXSWITCHER_UI_H
#define MUXSWITCHER_UI_H

#include "../lvgl/lvgl.h"
#include "switcher_model.h"

typedef enum {
    VIEW_NORMAL = 0,
    VIEW_MINIMAL = 1,
    VIEW_FULLSCREEN = -1
} ViewMode;

typedef struct {
    lv_obj_t *screen;
    lv_obj_t *header;
    lv_obj_t *footer;
    lv_obj_t *game_name_label;
    lv_obj_t *screenshot_img;
    lv_obj_t *battery_label;
    lv_obj_t *time_label;
    lv_obj_t *legend;
    lv_obj_t *brightness_bar;
    lv_obj_t *popup_menu;
} SwitcherUI;

void switcher_ui_init(SwitcherUI *ui);
void switcher_ui_update_game(SwitcherUI *ui, GameItem *game);
void switcher_ui_set_view_mode(SwitcherUI *ui, ViewMode mode);
void switcher_ui_show_header(SwitcherUI *ui, bool show);
void switcher_ui_show_footer(SwitcherUI *ui, bool show);
void switcher_ui_update_battery(SwitcherUI *ui, int percentage);
void switcher_ui_show_brightness(SwitcherUI *ui, int level);
void switcher_ui_cleanup(SwitcherUI *ui);

#endif
```

**Implementation:**
```c
#include "switcher_ui.h"
#include "../common/theme.h"

void switcher_ui_init(SwitcherUI *ui) {
    ui->screen = lv_obj_create(NULL);
    lv_obj_set_style_bg_color(ui->screen, lv_color_black(), 0);
    
    // Create header
    ui->header = lv_obj_create(ui->screen);
    lv_obj_set_size(ui->header, LV_PCT(100), 60);
    lv_obj_align(ui->header, LV_ALIGN_TOP_MID, 0, 0);
    lv_obj_set_style_bg_color(ui->header, theme.HEADER.BACKGROUND, 0);
    
    // Battery indicator
    ui->battery_label = lv_label_create(ui->header);
    lv_obj_align(ui->battery_label, LV_ALIGN_TOP_RIGHT, -10, 10);
    lv_obj_set_style_text_color(ui->battery_label, theme.HEADER.TEXT, 0);
    
    // Time label
    ui->time_label = lv_label_create(ui->header);
    lv_obj_align(ui->time_label, LV_ALIGN_TOP_LEFT, 10, 10);
    
    // Screenshot container
    ui->screenshot_img = lv_img_create(ui->screen);
    lv_obj_align(ui->screenshot_img, LV_ALIGN_CENTER, 0, 0);
    
    // Game name footer
    ui->footer = lv_obj_create(ui->screen);
    lv_obj_set_size(ui->footer, LV_PCT(100), 60);
    lv_obj_align(ui->footer, LV_ALIGN_BOTTOM_MID, 0, 0);
    lv_obj_set_style_bg_color(ui->footer, theme.FOOTER.BACKGROUND, 0);
    
    ui->game_name_label = lv_label_create(ui->footer);
    lv_obj_align(ui->game_name_label, LV_ALIGN_CENTER, 0, 0);
    lv_obj_set_style_text_font(ui->game_name_label, &lv_font_montserrat_24, 0);
    lv_label_set_long_mode(ui->game_name_label, LV_LABEL_LONG_SCROLL_CIRCULAR);
    
    // Brightness overlay (hidden by default)
    ui->brightness_bar = lv_bar_create(ui->screen);
    lv_obj_add_flag(ui->brightness_bar, LV_OBJ_FLAG_HIDDEN);
}

void switcher_ui_update_game(SwitcherUI *ui, GameItem *game) {
    // Update game name
    lv_label_set_text(ui->game_name_label, game->display_name);
    
    // Update screenshot
    if (game->screenshot) {
        lv_img_set_src(ui->screenshot_img, game->screenshot);
    }
    
    // Update playtime
    if (strlen(game->playtime) > 0) {
        lv_label_set_text(ui->time_label, game->playtime);
    }
}

void switcher_ui_set_view_mode(SwitcherUI *ui, ViewMode mode) {
    switch (mode) {
        case VIEW_FULLSCREEN:
            lv_obj_add_flag(ui->header, LV_OBJ_FLAG_HIDDEN);
            lv_obj_add_flag(ui->footer, LV_OBJ_FLAG_HIDDEN);
            break;
        case VIEW_MINIMAL:
            lv_obj_add_flag(ui->header, LV_OBJ_FLAG_HIDDEN);
            lv_obj_clear_flag(ui->footer, LV_OBJ_FLAG_HIDDEN);
            break;
        case VIEW_NORMAL:
            lv_obj_clear_flag(ui->header, LV_OBJ_FLAG_HIDDEN);
            lv_obj_clear_flag(ui->footer, LV_OBJ_FLAG_HIDDEN);
            break;
    }
}
```

#### Tests

```c
// test/switcher/test_ui.c (using LVGL simulator)

void test_ui_init_CreatesAllComponents(void) {
    SwitcherUI ui = {0};
    switcher_ui_init(&ui);
    
    TEST_ASSERT_NOT_NULL(ui.screen);
    TEST_ASSERT_NOT_NULL(ui.header);
    TEST_ASSERT_NOT_NULL(ui.footer);
    TEST_ASSERT_NOT_NULL(ui.game_name_label);
    
    switcher_ui_cleanup(&ui);
}

void test_ui_setViewMode_HidesFooterInFullscreen(void) {
    SwitcherUI ui = {0};
    switcher_ui_init(&ui);
    
    switcher_ui_set_view_mode(&ui, VIEW_FULLSCREEN);
    
    TEST_ASSERT_TRUE(lv_obj_has_flag(ui.footer, LV_OBJ_FLAG_HIDDEN));
    TEST_ASSERT_TRUE(lv_obj_has_flag(ui.header, LV_OBJ_FLAG_HIDDEN));
}
```

---

### **Phase 3: Input Handling (Week 6)**

#### Objectives
- Port keystate handling to muOS input system
- Implement long-press detection
- Add combo handlers (Menu button trigger)

#### Deliverables

**switcher_input.c:**
```c
#include "switcher_input.h"
#include "../common/input.h"

typedef struct {
    uint32_t press_start;
    bool is_held;
} InputState;

static InputState input_states[MUX_INPUT_COUNT] = {0};
static const uint32_t LONG_PRESS_THRESHOLD = 500; // ms

void switcher_input_handle_dpad_left(void) {
    // Navigate to previous game
    if (state->current_index > 0) {
        state->current_index--;
        switcher_ui_update_game(&ui, &state->items[state->current_index]);
    }
}

void switcher_input_handle_dpad_right(void) {
    // Navigate to next game
    if (state->current_index < state->count - 1) {
        state->current_index++;
        switcher_ui_update_game(&ui, &state->items[state->current_index]);
    }
}

void switcher_input_handle_a_press(void) {
    // Resume selected game
    GameItem *game = &state->items[state->current_index];
    switcher_launch_game(game);
}

void switcher_input_handle_b_press(void) {
    // Exit to menu
    state->quit = true;
    state->exit_to_menu = true;
}

void switcher_input_handle_x_press(void) {
    // Show remove confirmation dialog
    switcher_ui_show_confirm_dialog("Remove from history?");
}

void switcher_input_handle_y_long(void) {
    // Toggle fullscreen
    state->view_mode = (state->view_mode == VIEW_FULLSCREEN) 
                        ? state->view_restore 
                        : VIEW_FULLSCREEN;
    switcher_ui_set_view_mode(&ui, state->view_mode);
}

void switcher_input_handle_start_press(void) {
    // Open save state menu
    switcher_saves_show_menu();
}

void switcher_input_init(mux_input_options *opts) {
    // Register handlers
    opts->press_handlers[MUX_INPUT_DPAD_LEFT] = switcher_input_handle_dpad_left;
    opts->press_handlers[MUX_INPUT_DPAD_RIGHT] = switcher_input_handle_dpad_right;
    opts->press_handlers[MUX_INPUT_A] = switcher_input_handle_a_press;
    opts->press_handlers[MUX_INPUT_B] = switcher_input_handle_b_press;
    opts->press_handlers[MUX_INPUT_X] = switcher_input_handle_x_press;
    opts->press_handlers[MUX_INPUT_START] = switcher_input_handle_start_press;
    opts->hold_handlers[MUX_INPUT_Y] = switcher_input_handle_y_long;
}
```

#### Tests

```c
void test_input_LeftNav_DecrementsIndex(void) {
    SwitcherState state = {.count = 5, .current_index = 2};
    
    switcher_input_handle_dpad_left();
    
    TEST_ASSERT_EQUAL_INT(1, state.current_index);
}

void test_input_LeftNav_ClampAtZero(void) {
    SwitcherState state = {.count = 5, .current_index = 0};
    
    switcher_input_handle_dpad_left();
    
    TEST_ASSERT_EQUAL_INT(0, state.current_index);
}
```

---

### **Phase 4: RetroArch Integration (Weeks 7-8)**

#### Objectives
- Implement process-based control (SIGSTOP/SIGCONT)
- Add config parsing (cascading overrides)
- Create kill/restart mechanism for game switching
- Implement config-based save/load

#### Deliverables

**switcher_ra.h/c:**
```c
#ifndef MUXSWITCHER_RA_H
#define MUXSWITCHER_RA_H

#include <stdbool.h>
#include <signal.h>
#include <sys/types.h>

typedef enum {
    RA_CONTROL_PAUSE,      // SIGSTOP
    RA_CONTROL_RESUME,     // SIGCONT
    RA_CONTROL_SAVE_AUTO,  // Config-based auto-save
    RA_CONTROL_LOAD_AUTO,  // Config-based auto-load
    RA_CONTROL_KILL        // SIGKILL
} RAControlAction;

bool switcher_ra_control(RAControlAction action);
pid_t switcher_ra_find_process(void);
bool switcher_ra_is_running(void);
bool switcher_ra_parse_config(const char *core_name, const char *rom_name, char *aspect_ratio_out);
bool switcher_ra_find_core(const char *rom_path, char *core_path_out, char *core_name_out);
bool switcher_ra_write_autoload_config(bool enable);
bool switcher_ra_write_autosave_config(bool enable);

#endif
```

**Implementation:**
```c
#include "switcher_ra.h"
#include <signal.h>
#include <dirent.h>
#include <stdio.h>
#include <string.h>

pid_t switcher_ra_find_process(void) {
    // Check via muOS system variable first
    FILE *fp = popen("pgrep -x retroarch", "r");
    if (!fp) return -1;
    
    pid_t pid = -1;
    fscanf(fp, "%d", &pid);
    pclose(fp);
    
    return pid;
}

bool switcher_ra_is_running(void) {
    return switcher_ra_find_process() > 0;
}

bool switcher_ra_control(RAControlAction action) {
    pid_t ra_pid = switcher_ra_find_process();
    if (ra_pid <= 0) return false;
    
    switch (action) {
        case RA_CONTROL_PAUSE:
            // Pause emulation by stopping the process
            return kill(ra_pid, SIGSTOP) == 0;
            
        case RA_CONTROL_RESUME:
            // Resume emulation
            return kill(ra_pid, SIGCONT) == 0;
            
        case RA_CONTROL_SAVE_AUTO:
            // Write auto-save config that RA will pick up
            return switcher_ra_write_autosave_config(true);
            
        case RA_CONTROL_LOAD_AUTO:
            // Write auto-load config
            return switcher_ra_write_autoload_config(true);
            
        case RA_CONTROL_KILL:
            // Force quit RetroArch
            return kill(ra_pid, SIGKILL) == 0;
    }
    return false;
}

bool switcher_ra_write_autoload_config(bool enable) {
    const char *config_path = "/tmp/ra_autoload_once.cfg";
    FILE *fp = fopen(config_path, "w");
    if (!fp) return false;
    
    fprintf(fp, "savestate_auto_load = \"%s\"\n", enable ? "true" : "false");
    fclose(fp);
    
    return true;
}

bool switcher_ra_write_autosave_config(bool enable) {
    const char *config_path = "/tmp/ra_autosave.cfg";
    FILE *fp = fopen(config_path, "w");
    if (!fp) return false;
    
    fprintf(fp, "savestate_auto_save = \"%s\"\n", enable ? "true" : "false");
    fprintf(fp, "savestate_auto_index = \"true\"\n");
    fclose(fp);
    
    return true;
}

bool switcher_ra_parse_config(const char *core_name, const char *rom_name, 
                              char *aspect_ratio_out) {
    // Config cascade: game -> dir -> core -> global
    const char *config_paths[] = {
        "/mnt/mmc/MUOS/retroarch/config/%s/%s.cfg",      // Game-specific
        "/mnt/mmc/MUOS/retroarch/config/%s/config.cfg",  // Core-specific
        "/mnt/mmc/MUOS/retroarch/retroarch.cfg"          // Global
    };
    
    for (int i = 0; i < 3; i++) {
        char path[MAX_BUFFER_SIZE];
        snprintf(path, sizeof(path), config_paths[i], core_name, rom_name);
        
        if (file_exist(path)) {
            // Parse INI-style config
            char *value = config_get_value(path, "video_aspect_ratio");
            if (value) {
                strncpy(aspect_ratio_out, value, 32);
                free(value);
                return true;
            }
        }
    }
    
    strcpy(aspect_ratio_out, "auto");
    return false;
}
```

#### Tests

```c
void test_ra_control_ValidPause(void) {
    // Setup: Launch mock RetroArch process
    pid_t mock_pid = fork();
    if (mock_pid == 0) {
        // Child: pretend to be RetroArch
        while(1) sleep(1);
    }
    
    // Test pause
    bool result = switcher_ra_control(RA_CONTROL_PAUSE);
    TEST_ASSERT_TRUE(result);
    
    // Verify process is stopped
    char status[256];
    snprintf(status, sizeof(status), "/proc/%d/status", mock_pid);
    FILE *fp = fopen(status, "r");
    char line[256];
    bool is_stopped = false;
    while (fgets(line, sizeof(line), fp)) {
        if (strstr(line, "State:") && strstr(line, "T (stopped)")) {
            is_stopped = true;
            break;
        }
    }
    fclose(fp);
    TEST_ASSERT_TRUE(is_stopped);
    
    // Cleanup
    kill(mock_pid, SIGKILL);
}

void test_ra_writeAutoloadConfig_CreatesFile(void) {
    remove("/tmp/ra_autoload_once.cfg");
    
    bool result = switcher_ra_write_autoload_config(true);
    
    TEST_ASSERT_TRUE(result);
    TEST_ASSERT_TRUE(file_exist("/tmp/ra_autoload_once.cfg"));
    
    // Verify content
    char *content = read_file("/tmp/ra_autoload_once.cfg");
    TEST_ASSERT_TRUE(strstr(content, "savestate_auto_load = \"true\"") != NULL);
    free(content);
}

void test_ra_parseConfig_CascadePriority(void) {
    // Create test configs with different values
    create_test_config("game.cfg", "video_aspect_ratio = 16:9");
    create_test_config("core.cfg", "video_aspect_ratio = 4:3");
    
    char aspect[32];
    switcher_ra_parse_config("test_core", "test_game", aspect);
    
    TEST_ASSERT_EQUAL_STRING("16:9", aspect);  // Game config wins
}
```

---

### **Phase 5: Screenshot System (Week 9)**

#### Objectives
- Port screenshot caching system
- Implement FNV1A hashing for filenames
- Add lazy loading and LRU eviction
- Integrate framebuffer capture for overlay mode

#### Deliverables

**switcher_screenshots.c:**
```c
#include "switcher_screenshots.h"
#include "../common/common.h"
#include <png.h>

// FNV-1a hash (32-bit)
uint32_t fnv1a_hash(const char *str) {
    uint32_t hash = 2166136261u;
    while (*str) {
        hash ^= (uint8_t)*str++;
        hash *= 16777619u;
    }
    return hash;
}

char *switcher_screenshot_get_path(const char *rom_path, char *out_path, size_t size) {
    uint32_t hash = fnv1a_hash(rom_path);
    snprintf(out_path, size, "%s/%08x.png", ROM_SCREENS_DIR, hash);
    return out_path;
}

lv_img_dsc_t *switcher_screenshot_load(const char *rom_path) {
    char path[MAX_BUFFER_SIZE];
    switcher_screenshot_get_path(rom_path, path, sizeof(path));
    
    if (!file_exist(path)) {
        return NULL;
    }
    
    // Load PNG and convert to LVGL image descriptor
    lv_img_dsc_t *img = malloc(sizeof(lv_img_dsc_t));
    if (lv_img_decoder_open(img, path, LV_COLOR_FORMAT_ARGB8888) != LV_RES_OK) {
        free(img);
        return NULL;
    }
    
    return img;
}

bool switcher_screenshot_capture_framebuffer(const char *output_path) {
    // Read from /dev/fb0 and save as PNG
    int fb_fd = open("/dev/fb0", O_RDONLY);
    if (fb_fd < 0) return false;
    
    struct fb_var_screeninfo vinfo;
    ioctl(fb_fd, FBIOGET_VSCREENINFO, &vinfo);
    
    size_t screen_size = vinfo.xres * vinfo.yres * (vinfo.bits_per_pixel / 8);
    uint8_t *fb_data = mmap(NULL, screen_size, PROT_READ, MAP_SHARED, fb_fd, 0);
    
    // Write PNG
    FILE *fp = fopen(output_path, "wb");
    png_structp png = png_create_write_struct(PNG_LIBPNG_VER_STRING, NULL, NULL, NULL);
    png_infop info = png_create_info_struct(png);
    
    png_init_io(png, fp);
    png_set_IHDR(png, info, vinfo.xres, vinfo.yres, 8,
                 PNG_COLOR_TYPE_RGB, PNG_INTERLACE_NONE,
                 PNG_COMPRESSION_TYPE_DEFAULT, PNG_FILTER_TYPE_DEFAULT);
    png_write_info(png, info);
    
    // Write rows...
    for (int y = 0; y < vinfo.yres; y++) {
        png_write_row(png, fb_data + (y * vinfo.xres * 4));
    }
    
    png_write_end(png, NULL);
    fclose(fp);
    munmap(fb_data, screen_size);
    close(fb_fd);
    
    return true;
}
```

---

### **Phase 6: Save State Menu (Week 10)**

#### Objectives
- Create popup menu UI
- Implement state enumeration
- Add threading for save/load operations

**switcher_saves.c:**
```c
void switcher_saves_show_menu(void) {
    // Create popup
    lv_obj_t *popup = lv_obj_create(lv_scr_act());
    lv_obj_set_size(popup, LV_PCT(80), LV_PCT(70));
    lv_obj_center(popup);
    
    // List save states
    char states_path[MAX_BUFFER_SIZE];
    snprintf(states_path, sizeof(states_path), "%s/%s", 
             STATES_DIR, current_game->core_name);
    
    DIR *dir = opendir(states_path);
    if (dir) {
        struct dirent *entry;
        while ((entry = readdir(dir)) != NULL) {
            if (strstr(entry->d_name, ".state")) {
                // Add to list
                lv_obj_t *item = lv_list_add_btn(list, NULL, entry->d_name);
            }
        }
        closedir(dir);
    }
}

void *_save_thread(void *arg) {
    switcher_ra_send_command(RA_CMD_SAVE_STATE, state_slot);
    // Show progress
    return NULL;
}
```

---

### **Phase 7: Integration & Testing (Weeks 11-12)**

#### Objectives
- End-to-end testing
- Performance optimization
- Memory leak detection
- muOS launcher integration

#### Integration Tests

```c
// test/switcher/test_integration.c

void test_fullFlow_LaunchFromHistory(void) {
    // Setup: Create history file
    // Trigger: Launch gameSwitcher
    // Action: Navigate to game, press A
    // Verify: Game launch command written
}

void test_overlayMode_PauseResume(void) {
    // Setup: Start RetroArch with game
    // Trigger: Launch switcher --overlay
    // Verify: RetroArch paused
    // Action: Navigate, press B to resume
    // Verify: RetroArch unpaused
}

void test_saveState_ThreadedOperation(void) {
    // Setup: Game running
    // Action: Press Start, select Save
    // Verify: State file created
    // Verify: No UI blocking
}
```

---

## Technical Specifications

### Memory Management

**Allocation Strategy:**
- Static allocation for `SwitcherState` (< 100KB)
- Dynamic allocation for screenshots (freed after navigation)
- LRU cache: Keep 5 screenshots in memory (± 2 from current)

**Leak Prevention:**
```c
void switcher_cleanup(void) {
    // Free all screenshots
    for (int i = 0; i < state.count; i++) {
        if (state.items[i].screenshot) {
            lv_img_decoder_close(state.items[i].screenshot);
            free(state.items[i].screenshot);
        }
    }
    
    switcher_ui_cleanup(&ui);
}
```

### Threading Model

**Screenshot Loading Thread:**
```c
pthread_t screenshot_thread;

void *_load_screenshots_thread(void *arg) {
    for (int i = 0; i < 10 && i < state.count; i++) {
        state.items[i].screenshot = switcher_screenshot_load(
            state.items[i].recent_item.rompath
        );
    }
    return NULL;
}

pthread_create(&screenshot_thread, NULL, _load_screenshots_thread, NULL);
```

### Performance Targets

| Metric | Target | Measurement |
|--------|--------|-------------|
| Startup time | < 500ms | Time to first render |
| Navigation lag | < 16ms | Frame time (60 FPS) |
| Screenshot load | < 100ms | PNG decode + LVGL conversion |
| Memory footprint | < 20MB | RSS during operation |
| Save state time | < 2s | Thread completion |

---

## Integration Points

### muOS Launcher Integration

**Hotkey Trigger (muhotkey.c):**
```c
// Add to muhotkey.c combo handlers
static void handle_menu_long_press(void) {
    if (is_retroarch_running()) {
        system("muxswitcher --overlay &");
    }
}
```

**Frontend Integration (muxfrontend.c):**
```c
// Add to module table
ModuleEntry modules[] = {
    // ... existing modules ...
    {
        .action = "switcher",
        .module = "muxswitcher",
        .mux_main = muxswitcher_main
    }
};
```

### File System Paths (muOS-specific)

```c
#define MUOS_PLAYTIME_PATH "/mnt/mmc/MUOS/info/tracker/playtime_data.json"  // Custom JSON format
#define MUOS_ROM_SCREENS   "/mnt/mmc/MUOS/save/screenshots"  // TO BE VERIFIED
#define MUOS_STATES_DIR    "/mnt/mmc/MUOS/save/state"  // TO BE VERIFIED
#define MUOS_RA_CONFIG     "/mnt/mmc/MUOS/retroarch/retroarch.cfg"
#define MUOS_TEMP_FLAGS    "/tmp/muos"
#define MUOS_ROM_GO        "/tmp/rom_go"  // Game launch parameters
```

---

## Testing Strategy

### Unit Test Coverage Goals

| Component | Target Coverage | Priority |
|-----------|----------------|----------|
| History parsing | 95% | High |
| Input handling | 90% | High |
| RA integration | 85% | Medium |
| UI rendering | 70% | Medium |
| Screenshot system | 80% | Medium |

### Continuous Integration

**Build Pipeline:**
```yaml
# .github/workflows/switcher_tests.yml
name: GameSwitcher Tests

on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v2
      - name: Install dependencies
        run: |
          sudo apt-get install -y libsdl2-dev libpng-dev
          git clone https://github.com/lvgl/lvgl.git
      - name: Run tests
        run: |
          cd test/switcher
          make all
          ./run_tests
      - name: Coverage report
        run: lcov --capture --output-file coverage.info
```

### Manual Testing Checklist

- [ ] Launch from history (standalone mode)
- [ ] Launch from overlay (during gameplay)
- [ ] Navigate 100+ items without lag
- [ ] Screenshot loading/caching works
- [ ] Save state menu functional
- [ ] Load state from slot
- [ ] Delete state file
- [ ] Remove game from history
- [ ] Battery indicator updates
- [ ] Brightness adjustment works
- [ ] View mode switching (Y button)
- [ ] Graceful exit (B button)
- [ ] RetroArch pause/resume
- [ ] Hotkey trigger (Menu long-press)

---

## Risk Mitigation

### High-Risk Areas

| Risk | Impact | Mitigation |
|------|--------|----------|
| **LVGL performance** | UI lag | Profile early, use hardware acceleration |
| **RetroArch compatibility** | Feature broken | Mock RetroArch for testing |
| **Memory leaks** | System crash | Valgrind in CI, static analysis |
| **Threading deadlocks** | App freeze | Minimize shared state, use mutexes |
| **Device-specific bugs** | Hardware variance | Test on multiple muOS devices |

### Contingency Plans

**If LVGL performance is poor:**
- Fallback to simpler UI (no animations)
- Reduce screenshot resolution
- Limit concurrent image decoders

**If SIGSTOP/SIGCONT doesn't preserve state:**
- Fallback to kill/restart mechanism
- Use savestate auto-save before switching
- Implement config-based save/load triggers

---

## Success Criteria

### MVP (Minimum Viable Product)

- [x] Load history from file
- [x] Display game list with navigation
- [x] Launch selected game
- [x] Basic UI (name + screenshot)
- [x] Exit to menu

### Full Feature Parity

- [x] Overlay mode (pause RetroArch)
- [x] Save state menu
- [x] Screenshot caching
- [x] View modes (Normal/Minimal/Fullscreen)
- [x] Battery indicator
- [x] Brightness control
- [x] Hotkey trigger

### Performance Benchmarks

- [ ] < 500ms startup time
- [ ] 60 FPS navigation
- [ ] < 20MB memory usage
- [ ] No memory leaks after 1 hour

---

## Next Steps

1. **Set up test environment** (WSL2 + muOS build tools)
2. **Create project structure** (directories, Makefile)
3. **Write first test** (history parsing)
4. **Implement core data structures**
5. **Iterate through phases 1-7**

---

## References

- **OnionUI Source**: `reference/Onion/src/gameSwitcher/`
- **muOS Common Libraries**: `common/`
- **LVGL Documentation**: https://docs.lvgl.io/
- **RetroArch Network Commands**: https://docs.libretro.com/development/retroarch/network-control-interface/

---

**Document Version**: 1.0  
**Last Updated**: October 20, 2025  
**Author**: AI Assistant  
**Status**: Draft - Ready for Review
