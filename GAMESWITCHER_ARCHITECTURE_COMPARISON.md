# muOS vs OnionUI GameSwitcher Architecture Comparison

## Executive Summary

This document provides a detailed technical comparison between OnionUI's gameSwitcher implementation and the proposed muOS port. It serves as a reference for developers to understand the architectural differences and design decisions.

---

## Component-by-Component Analysis

### 1. Graphics Framework

#### OnionUI (SDL 1.2)

**Characteristics:**
- Immediate mode rendering (manual blitting)
- Software rendering (CPU-based)
- Direct pixel manipulation
- Manual memory management for surfaces

**Typical Rendering Pattern:**
```c
// OnionUI SDL 1.2 approach
SDL_Surface *screen = SDL_SetVideoMode(640, 480, 32, SDL_HWSURFACE);
SDL_Surface *image = IMG_Load("screenshot.png");

SDL_Rect dest = {x, y, 0, 0};
SDL_BlitSurface(image, NULL, screen, &dest);
SDL_Flip(screen);

SDL_FreeSurface(image);
```

**Pros:**
- Simple, predictable
- Direct framebuffer access
- Low memory overhead

**Cons:**
- Manual resource management
- No built-in layout system
- CPU-intensive for complex UIs
- No widget abstraction

---

#### muOS (LVGL)

**Characteristics:**
- Retained mode rendering (widget tree)
- Hardware acceleration support
- Automatic layout and styling
- Built-in memory management

**Equivalent Pattern:**
```c
// muOS LVGL approach
lv_obj_t *screen = lv_obj_create(NULL);
lv_obj_t *img = lv_img_create(screen);

lv_img_set_src(img, "screenshot.png");
lv_obj_align(img, LV_ALIGN_CENTER, 0, 0);

// Automatic rendering - no manual flip
// Automatic cleanup when screen destroyed
```

**Pros:**
- Widget abstraction (buttons, labels, images)
- Automatic dirty region tracking
- Built-in animations and transitions
- Declarative styling system
- Better memory safety

**Cons:**
- Higher learning curve
- More memory overhead
- Potential performance cost for simple UIs
- Less direct control

---

### **Migration Strategy**

**Abstraction Layer:**
```c
// Platform-agnostic interface
typedef struct {
    void *impl;  // SDL_Surface* or lv_obj_t*
    
    void (*set_image)(void *obj, const char *path);
    void (*set_position)(void *obj, int x, int y);
    void (*show)(void *obj, bool visible);
    void (*destroy)(void *obj);
} PlatformImage;

#ifdef PLATFORM_SDL
void platform_image_set(void *obj, const char *path) {
    SDL_Surface *surf = IMG_Load(path);
    // ... blitting logic
}
#endif

#ifdef PLATFORM_LVGL
void platform_image_set(void *obj, const char *path) {
    lv_img_set_src((lv_obj_t*)obj, path);
}
#endif
```

**Recommendation:** Full LVGL port (no abstraction layer) - muOS is LVGL-native, abstraction adds unnecessary complexity.

---

## 2. Input Handling

### OnionUI

**Architecture:**
```
evdev raw input → keymon daemon → gameSwitcher
                                    ↓
                            gs_keystate.h (polling)
```

**Input Loop:**
```c
// OnionUI approach
SDL_Event event;
while (SDL_PollEvent(&event)) {
    if (event.type == SDL_KEYDOWN) {
        switch (event.key.keysym.sym) {
            case SDLK_LEFT:
                navigate_left();
                break;
            case SDLK_RIGHT:
                navigate_right();
                break;
        }
    }
}
```

**Long-Press Detection:**
```c
uint32_t key_press_start[MAX_KEYS];

if (event.type == SDL_KEYDOWN && !key_held[key]) {
    key_press_start[key] = SDL_GetTicks();
    key_held[key] = true;
}

if (key_held[key] && SDL_GetTicks() - key_press_start[key] > LONG_PRESS_MS) {
    handle_long_press(key);
}
```

---

### muOS

**Architecture:**
```
evdev raw input → muhotkey daemon → mux module
                                      ↓
                              common/input.h (callback)
```

**Input Registration:**
```c
// muOS approach
mux_input_options opts = {0};

// Register callbacks
opts.press_handlers[MUX_INPUT_DPAD_LEFT] = handle_left;
opts.press_handlers[MUX_INPUT_DPAD_RIGHT] = handle_right;
opts.hold_handlers[MUX_INPUT_Y] = handle_y_long;

init_input(&opts, 0);

// Input processed automatically by common/input.c
// Callbacks invoked on events
```

**Long-Press Handling:**
```c
// Built into muOS input system
void handle_y_long(void) {
    // Automatically called after hold threshold
    toggle_fullscreen();
}
```

---

### **Comparison**

| Feature | OnionUI (SDL) | muOS (mux_input) |
|---------|---------------|------------------|
| **Event Model** | Polling (manual loop) | Callback (event-driven) |
| **Long Press** | Manual timer tracking | Built-in threshold detection |
| **Button Combos** | Manual bitmask | `mux_input_combo` struct |
| **Hotkeys** | Handled in keymon | Handled in muhotkey |
| **Integration** | `SDL_PollEvent()` | `init_input()` + callbacks |

**Recommendation:** Use muOS input system directly - provides all needed features with better integration.

---

## 3. State Management

### OnionUI

**State Storage:**
```c
// gs_appState.h - Global mutable state
typedef struct {
    int current_game;
    bool quit;
    bool changed;
    int view_mode;
    uint32_t acc_ticks;
    SDL_Surface *current_bg;
    bool show_legend;
    bool pop_menu_open;
    // ... 20+ fields
} AppState;

static AppState appState = {0};  // Global
```

**Communication:**
- Flag files in `/mnt/SDCARD/.tmp_update/`
- Shell scripts (`cmd_to_run.sh`)
- Process name checking

---

### muOS

**State Storage:**
```c
// Localized to module
typedef struct {
    GameItem items[MAX_HISTORY];
    int count;
    int current_index;
    bool quit;
    ViewMode view_mode;
} SwitcherState;

// In muxswitcher.c
int muxswitcher_main(void) {
    SwitcherState state = {0};  // Local
    // ... use state locally
}
```

**Communication:**
- Shared memory (potentially)
- muOS config system
- Direct function calls (same process space)

---

### **Comparison**

| Aspect | OnionUI | muOS |
|--------|---------|------|
| **Scope** | Global static state | Module-local state |
| **IPC** | Files + system() calls | Config API |
| **Launcher** | Shell script chains | Module function table |
| **Persistence** | Flag files | Config files |

**Recommendation:** Use module-local state with muOS config API for persistence. Cleaner than flag files.

---

## 4. History File Format

### Format Compatibility

**Good News:** Both OnionUI and muOS likely use RetroArch's standard `content_history.lpl` format.

**File Structure:**
```json
{"type":5,"label":"Super Mario World","core_path":"/path/to/core.so","core_name":"SNES9x","db_name":"Nintendo - Super Nintendo Entertainment System.lpl","path":"/roms/snes/smw.sfc","crc32":"FFFFFFFF","runtime_hours":12,"runtime_minutes":34,"runtime_seconds":56,"last_played":1697800000}
```

**Parsing (Identical):**
```c
// Both use cJSON
cJSON *json = cJSON_Parse(line);
cJSON *type = cJSON_GetObjectItem(json, "type");
cJSON *path = cJSON_GetObjectItem(json, "path");
```

**Key Fields:**

| Field | Purpose | Required |
|-------|---------|----------|
| `type` | Entry type (5 = game, 17 = recent) | ✓ |
| `label` | Display name | ✓ |
| `path` | ROM file path | ✓ |
| `core_path` | Core .so file | ✓ |
| `core_name` | Core display name |  |
| `last_played` | Unix timestamp |  |
| `runtime_*` | Playtime tracking |  |

---

### **Differences to Handle**

**OnionUI Specifics:**
- Custom `:` separator in rompath field for launch commands
- Type 17 for "recent" vs type 5 for "favorite"
- Imgpath for artwork fallback

**muOS Adjustments:**
```c
// Handle OnionUI colon separator
char *colon = strchr(item->rompath, ':');
if (colon) {
    // Split into launch command and path
    *colon = '\0';
    strcpy(item->launch, item->rompath);
    strcpy(item->rompath, colon + 1);
}
```

**Recommendation:** Add compatibility layer for OnionUI history files if users migrate.

---

## 5. RetroArch Integration

### Communication Methods

#### OnionUI (UDP Network Protocol)

**Connection:**
```c
#define RETROARCH_PORT 55355

int sock = socket(AF_INET, SOCK_DGRAM, 0);
struct sockaddr_in addr = {
    .sin_family = AF_INET,
    .sin_port = htons(RETROARCH_PORT),
    .sin_addr.s_addr = inet_addr("127.0.0.1")
};

sendto(sock, "PAUSE", 5, 0, (struct sockaddr*)&addr, sizeof(addr));
```

**Commands:**
- `PAUSE` - Pause emulation
- `UNPAUSE` - Resume
- `SAVE_STATE_SLOT N` - Set save slot
- `SAVE_STATE` - Save to current slot
- `LOAD_STATE_SLOT N` - Set load slot
- `LOAD_STATE` - Load from current slot
- `QUIT` - Exit RetroArch

---

#### muOS (Check Implementation)

**Need to Verify:**
```bash
# Check if muOS uses same UDP protocol
grep -r "55355\|retroarch.*port" /mnt/mmc/MUOS/
```

**Alternative Methods:**
1. **Unix sockets** - Local IPC
2. **Signals** - SIGUSR1/SIGUSR2
3. **Shared memory** - State exchange
4. **Config files** - Write-then-notify

---

### Config Cascading

**Both systems use same priority:**

```
1. Game-specific:     config/{core}/{rom_name}.cfg
2. Content directory: config/{core}/{dir_name}.cfg  
3. Core override:     config/{core}/{core}.cfg
4. Global:            retroarch.cfg
```

**Parsing Logic (Identical):**
```c
char *get_config_value(const char *core, const char *rom, const char *key) {
    char paths[4][MAX_PATH];
    snprintf(paths[0], MAX_PATH, "%s/%s/%s.cfg", CONFIG_DIR, core, rom);
    snprintf(paths[1], MAX_PATH, "%s/%s/%s.cfg", CONFIG_DIR, core, dirname(rom));
    snprintf(paths[2], MAX_PATH, "%s/%s/%s.cfg", CONFIG_DIR, core, core);
    snprintf(paths[3], MAX_PATH, "%s/retroarch.cfg", CONFIG_DIR);
    
    for (int i = 0; i < 4; i++) {
        char *value = ini_parse(paths[i], key);
        if (value) return value;
    }
    return NULL;
}
```

**Use Cases:**
- Aspect ratio settings (for screenshot scaling)
- Integer scaling preference
- Input mappings

---

## 6. Screenshot System

### Caching Strategy

#### OnionUI

**Hash-Based Filenames:**
```c
// FNV-1a 32-bit hash
uint32_t hash = 2166136261u;
for (const char *p = rom_path; *p; p++) {
    hash ^= (uint8_t)*p;
    hash *= 16777619u;
}

char filename[64];
snprintf(filename, sizeof(filename), "%08x.png", hash);
```

**Storage:**
```
/mnt/SDCARD/Saves/CurrentProfile/romScreens/
├── 00abcdef.png  (hash of /roms/nes/mario.nes)
├── 01234567.png
└── ...
```

**Pros:**
- Unique per ROM path
- Collision-resistant
- Fast lookup

---

#### muOS

**Likely Same Approach:**
```
/mnt/mmc/MUOS/save/screenshots/
├── {hash}.png
└── ...
```

**Verify with:**
```bash
ls -la /mnt/mmc/MUOS/save/screenshots/
```

---

### Loading & Caching

**OnionUI Strategy:**
- Load first 10 on startup (background thread)
- Lazy load on navigation
- LRU eviction: Keep current ± 5 in memory
- Unload screenshots beyond range

**Implementation:**
```c
void loadRomScreens() {
    pthread_t thread;
    pthread_create(&thread, NULL, _load_thread, NULL);
}

void *_load_thread(void *arg) {
    for (int i = 0; i < 10 && i < game_list_len; i++) {
        game_list[i].romScreen = loadRomScreen(i);
    }
    return NULL;
}

// On navigation
void on_game_change(int new_index) {
    // Unload far away screenshots
    for (int i = 0; i < game_list_len; i++) {
        if (abs(i - new_index) > 5) {
            if (game_list[i].romScreen) {
                SDL_FreeSurface(game_list[i].romScreen);
                game_list[i].romScreen = NULL;
            }
        }
    }
    
    // Preload nearby
    for (int i = new_index - 2; i <= new_index + 2; i++) {
        if (i >= 0 && i < game_list_len && !game_list[i].romScreen) {
            game_list[i].romScreen = loadRomScreen(i);
        }
    }
}
```

---

### Framebuffer Capture (Overlay Mode)

**OnionUI:**
```c
void setFbAsFirstRomScreen() {
    screenshot_system();  // Capture /dev/fb0
    
    // Read framebuffer
    int fb_fd = open("/dev/fb0", O_RDONLY);
    struct fb_var_screeninfo vinfo;
    ioctl(fb_fd, FBIOGET_VSCREENINFO, &vinfo);
    
    size_t size = vinfo.xres * vinfo.yres * (vinfo.bits_per_pixel / 8);
    uint8_t *fb_data = mmap(NULL, size, PROT_READ, MAP_SHARED, fb_fd, 0);
    
    // Convert to PNG and save
    save_as_png(fb_data, vinfo.xres, vinfo.yres, output_path);
}
```

**muOS Equivalent:**
```c
// Option 1: Direct framebuffer (if available)
void muos_capture_framebuffer(const char *output) {
    // Same as OnionUI
}

// Option 2: LVGL screenshot API
void muos_capture_lvgl_screen(const char *output) {
    lv_obj_t *screen = lv_scr_act();
    lv_snapshot_take(screen, output);
}

// Option 3: SDL2 (if muOS uses SDL2 for LVGL driver)
void muos_capture_sdl_screen(const char *output) {
    SDL_Surface *surface = SDL_GetWindowSurface(window);
    IMG_SavePNG(surface, output);
}
```

**Recommendation:** Test muOS's LVGL driver - if SDL2-based, use SDL_SavePNG. Otherwise, framebuffer direct.

---

## 7. Threading Model

### OnionUI

**Threads Used:**
1. **Main thread** - UI rendering, input handling
2. **Screenshot loader** - Background loading (startup)
3. **Save state thread** - Save operations (popup menu)
4. **Auto-save thread** - Save on overlay init

**Synchronization:**
```c
pthread_t save_thread;
pthread_mutex_t save_mutex = PTHREAD_MUTEX_INITIALIZER;

void *_save_thread(void *arg) {
    pthread_mutex_lock(&save_mutex);
    retroarch_save(slot);
    pthread_mutex_unlock(&save_mutex);
    return NULL;
}

void wait_for_save() {
    pthread_join(save_thread, NULL);
}
```

---

### muOS

**Recommended Threading:**

```c
typedef struct {
    pthread_t thread;
    pthread_mutex_t mutex;
    bool running;
    volatile bool cancel;
} ThreadContext;

// Long-running operations
ThreadContext screenshot_loader;
ThreadContext save_state_worker;

void start_background_loader(void) {
    screenshot_loader.cancel = false;
    screenshot_loader.running = true;
    pthread_create(&screenshot_loader.thread, NULL, load_screenshots, NULL);
}

void stop_background_loader(void) {
    screenshot_loader.cancel = true;
    pthread_join(screenshot_loader.thread, NULL);
    screenshot_loader.running = false;
}
```

**LVGL Thread Safety:**
```c
// LVGL is NOT thread-safe by default
// Must use mutex when updating UI from background thread

pthread_mutex_t lvgl_mutex = PTHREAD_MUTEX_INITIALIZER;

void *background_task(void *arg) {
    // Do work...
    
    // Update UI
    pthread_mutex_lock(&lvgl_mutex);
    lv_label_set_text(label, "Done!");
    pthread_mutex_unlock(&lvgl_mutex);
}

// In main loop
void main_loop() {
    while (running) {
        pthread_mutex_lock(&lvgl_mutex);
        lv_task_handler();  // LVGL update
        pthread_mutex_unlock(&lvgl_mutex);
        usleep(16000);  // 60 FPS
    }
}
```

---

## 8. Save State Management

### File Structure

**Both Systems:**
```
states/
├── {core_name}/
│   ├── {rom_name}.state      # Slot -1 (auto)
│   ├── {rom_name}.state0     # Slot 0
│   ├── {rom_name}.state1     # Slot 1
│   ├── {rom_name}.state0.png # Preview for slot 0
│   └── ...
```

**Enumeration:**
```c
void scan_save_states(const char *core, const char *rom) {
    char dir[MAX_PATH];
    snprintf(dir, sizeof(dir), "%s/%s", STATES_DIR, core);
    
    DIR *d = opendir(dir);
    if (!d) return;
    
    struct dirent *entry;
    while ((entry = readdir(d)) != NULL) {
        // Check if filename starts with rom name
        if (strncmp(entry->d_name, rom, strlen(rom)) == 0) {
            if (strstr(entry->d_name, ".state")) {
                // Parse slot number
                int slot = parse_slot_from_filename(entry->d_name);
                // Add to list
            }
        }
    }
    closedir(d);
}
```

---

### Save/Load Operations

**OnionUI:**
```c
void action_saveGame() {
    pthread_create(&save_thread, NULL, _save_thread, NULL);
}

void *_save_thread(void *arg) {
    // Find next available slot
    int slot = find_next_slot();
    
    // Tell RetroArch to save
    retroarch_save(slot);
    
    // Wait for file to appear
    wait_for_state_file(slot);
    
    // Show success message
    return NULL;
}
```

**muOS Equivalent:**
```c
void switcher_save_state(void) {
    int slot = find_next_available_slot();
    
    // Send UDP command
    switcher_ra_send_command(RA_CMD_SAVE_STATE_SLOT, slot);
    switcher_ra_send_command(RA_CMD_SAVE_STATE, 0);
    
    // Show progress UI
    switcher_ui_show_progress("Saving state...");
    
    // Poll for completion
    char state_file[MAX_PATH];
    get_state_path(slot, state_file);
    
    for (int i = 0; i < 50; i++) {  // 5 second timeout
        if (file_exist(state_file)) {
            switcher_ui_show_toast("State saved!");
            return;
        }
        usleep(100000);  // 100ms
    }
    
    switcher_ui_show_error("Save failed");
}
```

---

## 9. Memory Management

### OnionUI (SDL)

**Surface Lifecycle:**
```c
SDL_Surface *screen = SDL_SetVideoMode(...);      // Created once
SDL_Surface *image = IMG_Load("file.png");        // Allocated
SDL_BlitSurface(image, NULL, screen, &dest);      // Used
SDL_FreeSurface(image);                           // Freed manually
```

**Leaks to Avoid:**
- Forgetting `SDL_FreeSurface()`
- Not clearing game list on exit
- Dangling pointers after navigation

---

### muOS (LVGL)

**Widget Lifecycle:**
```c
lv_obj_t *screen = lv_obj_create(NULL);           // Created
lv_obj_t *img = lv_img_create(screen);            // Child of screen
lv_img_set_src(img, "file.png");                  // LVGL loads internally
lv_obj_del(screen);                                // Deletes children too!
```

**Automatic Management:**
- Parent-child relationship → cascade delete
- Image cache managed by LVGL
- Styles reference-counted

**Manual Management Still Needed:**
```c
// Custom image descriptors
lv_img_dsc_t *img = malloc(sizeof(lv_img_dsc_t));
lv_img_decoder_open(img, path, LV_COLOR_FORMAT_ARGB8888);
lv_img_set_src(obj, img);

// Later: must free manually
lv_img_decoder_close(img);
free(img);
```

---

### Valgrind Testing

**OnionUI Results (Example):**
```
==12345== LEAK SUMMARY:
==12345==    definitely lost: 512 bytes in 4 blocks
==12345==    indirectly lost: 2,048 bytes in 16 blocks
==12345==    still reachable: 8,192 bytes in 64 blocks
```

**Common Leaks:**
- `cJSON_Parse()` without `cJSON_Delete()`
- `IMG_Load()` without `SDL_FreeSurface()`
- `strdup()` without `free()`

**muOS Testing:**
```bash
valgrind --leak-check=full --track-origins=yes ./muxswitcher
```

---

## 10. Platform-Specific Differences

### Device Paths

| Resource | OnionUI | muOS (Assumed) |
|----------|---------|----------------|
| **SD Card** | `/mnt/SDCARD` | `/mnt/mmc` |
| **RetroArch** | `/mnt/SDCARD/RetroArch` | `/mnt/mmc/MUOS/retroarch` |
| **Saves** | `/mnt/SDCARD/Saves` | `/mnt/mmc/MUOS/save` |
| **Config** | `/mnt/SDCARD/.tmp_update` | `/tmp/muos` or `/mnt/mmc/MUOS/config` |
| **Framebuffer** | `/dev/fb0` | `/dev/fb0` (likely same) |

**Abstraction:**
```c
// config/paths.h
#ifdef PLATFORM_ONION
    #define STORAGE_ROOT "/mnt/SDCARD"
#elif defined(PLATFORM_MUOS)
    #define STORAGE_ROOT "/mnt/mmc/MUOS"
#endif

#define HISTORY_PATH STORAGE_ROOT "/save/history/content_history.lpl"
```

---

### Display Resolution

**OnionUI (Miyoo Mini):**
- 640x480 (default)
- 752x560 (A30 device)

**muOS (Multiple Devices):**
- Check `device.MUX.WIDTH` and `device.MUX.HEIGHT`
- Use `mux_dimension` string

**Responsive Design:**
```c
void create_ui(void) {
    int header_height = (int)(device.MUX.HEIGHT * 0.1);  // 10% of screen
    int footer_height = (int)(device.MUX.HEIGHT * 0.1);
    int content_height = device.MUX.HEIGHT - header_height - footer_height;
    
    lv_obj_set_height(ui_header, header_height);
    lv_obj_set_height(ui_footer, footer_height);
}
```

---

## Performance Comparison

### Rendering Speed

**SDL 1.2 (OnionUI):**
- Software rendering
- ~30-60 FPS typical
- Full screen blit every frame

**LVGL (muOS):**
- Dirty region tracking
- ~60 FPS with optimization
- Only redraws changed areas

**Optimization Tips:**

```c
// LVGL: Reduce redraws
lv_obj_add_flag(static_obj, LV_OBJ_FLAG_IGNORE_LAYOUT);
lv_obj_invalidate(obj);  // Mark dirty only when needed

// Disable animations for performance
lv_obj_set_style_anim_time(obj, 0, 0);
```

---

### Memory Footprint

**OnionUI:**
- Base: ~5MB (SDL + libraries)
- Screenshots: ~2MB each (640x480 RGBA)
- Total: ~15-20MB typical

**muOS:**
- Base: ~8MB (LVGL + SDL2 + libraries)
- Screenshots: ~2MB each
- Widget overhead: ~1-2MB
- Total: ~18-25MB typical

---

## Migration Checklist

### Code Audit

- [ ] Identify all SDL-specific code
- [ ] Map SDL functions to LVGL equivalents
- [ ] List all file paths (make configurable)
- [ ] Document RetroArch integration points
- [ ] Review threading (LVGL thread safety)
- [ ] Check memory allocation patterns

### Testing Requirements

- [ ] Unit tests for each component
- [ ] Integration test with mock RetroArch
- [ ] UI test with LVGL simulator
- [ ] Memory leak test (valgrind)
- [ ] Performance benchmark (60 FPS target)
- [ ] Device-specific testing (multiple muOS devices)

### Documentation Needed

- [ ] API reference for each module
- [ ] Integration guide for muOS launcher
- [ ] User guide (controls, features)
- [ ] Developer guide (architecture, extending)
- [ ] Troubleshooting guide

---

## Conclusion

### Key Takeaways

1. **~60% of code is portable** (history parsing, data structures, RetroArch protocol)
2. **~30% needs rewriting** (SDL → LVGL rendering)
3. **~10% needs adaptation** (paths, input system, muOS integration)

### Estimated Effort Breakdown

| Phase | Effort | Dependencies |
|-------|--------|--------------|
| Data structures | 3 days | None |
| History parsing | 5 days | cJSON |
| LVGL UI | 15 days | LVGL knowledge |
| Input system | 3 days | muOS input API |
| RetroArch integration | 7 days | UDP testing |
| Screenshot system | 5 days | PNG library |
| Save state menu | 5 days | Threading |
| Testing & debugging | 10 days | All components |
| **Total** | **~53 days** | ~10-12 weeks |

### Critical Success Factors

✓ **muOS must support RetroArch** (verify UDP protocol or alternative)  
✓ **LVGL performance** (test on target hardware early)  
✓ **Framebuffer access** (for overlay mode screenshots)  
✓ **Input system compatibility** (hotkey trigger from muhotkey)  
✓ **Memory constraints** (< 25MB footprint)

---

**Document Version**: 1.0  
**Created**: October 20, 2025  
**Purpose**: Technical reference for gameSwitcher port
