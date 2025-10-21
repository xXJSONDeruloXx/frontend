# muOS GameSwitcher Project - Analysis Summary

## Overview

This document summarizes the comprehensive analysis of porting OnionUI's gameSwitcher feature to muOS (MustardOS). After examining both the OnionUI implementation and the muOS internal codebase, we now have a clear roadmap for implementation.

---

## Key Documents Created

1. **`GAMESWITCHER_IMPLEMENTATION_PLAN.md`**
   - Comprehensive TDD implementation plan
   - 12-phase development roadmap (8-12 weeks estimate)
   - Code examples and test cases for each component
   - Memory management and threading strategies

2. **`GAMESWITCHER_ARCHITECTURE_COMPARISON.md`**
   - Detailed SDL 1.2 → LVGL migration guide
   - Component-by-component comparison (OnionUI vs muOS)
   - Performance analysis and optimization recommendations
   - 53-day effort breakdown

3. **`GAMESWITCHER_ANSWERED_QUESTIONS.md`**
   - Resolved unknowns from internal repository analysis
   - Critical implementation adjustments based on findings
   - Hardware validation checklist
   - Risk assessment and mitigation strategies

---

## Major Findings

### ✅ What We Know

1. **RetroArch Communication**
   - muOS uses **process-based control** (NOT UDP like OnionUI)
   - Must use SIGSTOP/SIGCONT for pause/resume
   - Config manipulation for save/load states
   - Kill/restart approach for game switching

2. **History Management**
   - muOS uses **custom JSON format** (`playtime_data.json`)
   - NOT using RetroArch's `content_history.lpl`
   - Rich playtime analytics already integrated
   - Tracked per-core, per-device, per-mode

3. **File System Layout**
   - Root: `/mnt/mmc/MUOS` (vs OnionUI's `/mnt/SDCARD`)
   - History: `/mnt/mmc/MUOS/info/track/playtime_data.json`
   - Saves: `/mnt/mmc/MUOS/save/state/{core}/{rom}`
   - RetroArch config: `/mnt/mmc/MUOS/info/config/retroarch.cfg`

4. **Module System**
   - Uses `EXEC_MUX()` for module execution
   - `/tmp/rom_go` file for game launch parameters
   - Integrated with `muhotkey` for global hotkeys
   - Standard frontend lifecycle with `muxfrontend`

5. **Technology Stack**
   - C language
   - LVGL for UI (version in frontend repo)
   - SDL2 as LVGL driver (likely)
   - cJSON for JSON parsing
   - Shell scripts for system integration

### ⚠️ What Needs Hardware Validation

1. **RetroArch Pause Mechanism**
   - Will `kill -STOP` preserve emulation state?
   - Does framebuffer remain accessible when paused?
   - Can we resume without corruption?

2. **Screenshot Storage**
   - Where does muOS store screenshots?
   - Are they auto-generated or manual?
   - What naming convention is used?

3. **LVGL Driver**
   - SDL2, framebuffer, or DRM?
   - Performance characteristics?
   - Screenshot capture method?

4. **RetroArch History**
   - Does `content_history.lpl` exist alongside `playtime_data.json`?
   - Can we leverage both sources?

5. **Save State Control**
   - Can config-based save/load work without UDP?
   - Will RetroArch reload configs mid-execution?

---

## Adjusted Architecture

### Core Components

```
muxswitcher/
├── muxswitcher.c                   # Main module entry point
├── switcher_model.h                # Data structures
├── switcher_history.{c,h}          # JSON history parsing
├── switcher_ui.{c,h}               # LVGL UI components
├── switcher_input.{c,h}            # muOS input integration
├── switcher_ra.{c,h}               # Process-based RA control
├── switcher_screenshots.{c,h}      # Screenshot management
└── switcher_saves.{c,h}            # Save state menu
```

### Key Differences from OnionUI

| Feature | OnionUI | muOS Implementation |
|---------|---------|---------------------|
| **Graphics** | SDL 1.2 | LVGL + SDL2 |
| **RA Control** | UDP commands | Process signals |
| **History** | content_history.lpl | playtime_data.json |
| **Paths** | /mnt/SDCARD | /mnt/mmc/MUOS |
| **State Management** | Flag files | muOS config vars |
| **Hotkeys** | keymon | muhotkey |

---

## Implementation Timeline

### Phase 1: Setup & Validation (Week 1)
- [ ] Set up muOS build environment
- [ ] Test critical unknowns on hardware
- [ ] Create proof of concept (simple LVGL window)

### Phase 2: Core Data (Week 2)
- [ ] Parse `playtime_data.json`
- [ ] Create GameItem structures
- [ ] Implement history deduplication

### Phase 3: LVGL UI (Weeks 3-4)
- [ ] Build game list widget
- [ ] Create header/footer components
- [ ] Add screenshot display
- [ ] Implement view modes

### Phase 4: RetroArch Control (Week 5)
- [ ] Process-based pause/resume
- [ ] Config-based state save/load
- [ ] Kill/restart game switching

### Phase 5: Input & Navigation (Week 6)
- [ ] muOS input system integration
- [ ] Navigation handlers
- [ ] Long-press detection
- [ ] Hotkey trigger in muhotkey

### Phase 6: Screenshots (Week 7)
- [ ] Screenshot loading system
- [ ] Framebuffer capture
- [ ] LRU caching
- [ ] Memory optimization

### Phase 7: Save State Menu (Week 8)
- [ ] Popup UI
- [ ] State file enumeration
- [ ] Preview images
- [ ] Delete functionality

### Phase 8: Testing & Polish (Weeks 9-12)
- [ ] Hardware testing
- [ ] Performance profiling
- [ ] Memory leak fixes
- [ ] Documentation

---

## Risk Assessment

### Critical Risks

| Risk | Mitigation |
|------|------------|
| **SIGSTOP doesn't work** | Fall back to kill/restart approach |
| **No content_history.lpl** | Use playtime_data.json exclusively |
| **LVGL performance issues** | Optimize rendering, reduce animations |
| **Limited save state control** | Document limitations, focus on switching |

### Assumptions

✅ **Safe Assumptions:**
- muOS uses process-based RA control
- History tracking is custom JSON
- LVGL is the UI framework
- muhotkey exists and is extensible

⚠️ **Requires Validation:**
- SIGSTOP preserves RA state
- Screenshot directory structure
- LVGL driver type
- Save state manipulation method

---

## Success Metrics

### Minimum Viable Product (MVP)
- Load history from muOS JSON
- Display navigable game list
- Launch selected game
- Basic LVGL UI
- Hotkey integration

### Feature Complete
- Pause/resume (if signals work)
- Screenshot display
- Playtime information
- Save state menu (if config method works)
- View mode switching

### Optimal Experience
- Sub-500ms startup
- 60 FPS navigation
- < 20MB memory footprint
- No memory leaks
- Full theme support

---

## Next Actions

### Immediate (This Week)
1. ✅ Clone muOS internal repository - **DONE**
2. ✅ Analyze RetroArch integration - **DONE**
3. ✅ Document findings - **DONE**
4. ⏭️ Set up muOS build environment
5. ⏭️ Create proof of concept

### Short Term (Weeks 2-4)
6. Test SIGSTOP/SIGCONT on hardware
7. Verify screenshot locations
8. Implement history parser
9. Build basic LVGL UI

### Medium Term (Weeks 5-8)
10. Implement RA control layer
11. Add screenshot system
12. Create save state menu
13. Integrate with muhotkey

### Long Term (Weeks 9-12)
14. Hardware testing campaign
15. Performance optimization
16. Bug fixes and polish
17. Documentation and release

---

## Questions for muOS Maintainers

If you have access to muOS developers, ask:

1. **RetroArch Control**
   - How do you currently pause/resume RetroArch?
   - Is there an API for triggering save states?
   - Any plans for UDP command support?

2. **Screenshots**
   - Where are ROM screenshots stored?
   - Are they auto-generated or user-created?
   - What's the preferred capture method?

3. **History**
   - Do you maintain `content_history.lpl` for RA compatibility?
   - Can we extend `playtime_data.json` structure?
   - Any plans for game switching feature?

4. **Integration**
   - Best practice for new mux modules?
   - How to extend muhotkey combos?
   - Module testing framework exists?

---

## References

### Codebases
- **OnionUI**: `reference/Onion/src/gameSwitcher/`
- **muOS Frontend**: `frontend/` (this repo)
- **muOS Internal**: `reference/internal/`

### Key Files
- `reference/Onion/src/gameSwitcher/gameSwitcher.c` - Original implementation
- `reference/internal/script/var/func.sh` - muOS utilities
- `reference/internal/script/mux/track.sh` - Playtime tracking
- `reference/internal/script/launch/lr-general.sh` - RetroArch launcher

### Documentation
- LVGL: https://docs.lvgl.io/
- RetroArch: https://docs.libretro.com/
- cJSON: https://github.com/DaveGamble/cJSON

---

## Conclusion

**Feasibility: HIGH** - The gameSwitcher feature is definitely portable to muOS with the identified adjustments.

**Confidence: 80%** - Most critical questions answered; remaining 20% requires hardware validation.

**Estimated Effort: 10-12 weeks** - For full feature parity with OnionUI implementation.

**Biggest Challenge: RetroArch Control** - Process-based approach is untested; may need iteration.

**Biggest Opportunity: Better Integration** - muOS's playtime tracking is more sophisticated than OnionUI's; can leverage existing infrastructure.

---

**Ready to Begin Implementation** ✅

The analysis phase is complete. All architectural decisions are documented. Test-driven development plan is in place. Proceed to Phase 1: Hardware Validation & Proof of Concept.

---

**Document Version**: 1.0  
**Date**: October 20, 2025  
**Author**: AI Assistant  
**Status**: Analysis Complete - Implementation Ready
