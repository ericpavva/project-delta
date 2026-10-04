do

local VERSION = "2.8.4"

local UI = {}
UI.__index = UI

UI.COLORS_DEFAULT = {
    bg          = {0.05, 0.06, 0.10, 0.96},
    titlebar    = {0.08, 0.10, 0.16, 1.00},
    border      = {0.16, 0.22, 0.34, 1.00},
    border_soft = {0.12, 0.16, 0.24, 1.00},
    accent      = {0.25, 0.60, 1.00, 1.00},
    accent_dim  = {0.25, 0.60, 1.00, 0.25},
    text        = {0.92, 0.95, 1.00, 1.00},
    text_dim    = {0.55, 0.60, 0.72, 1.00},
    text_muted  = {0.35, 0.40, 0.50, 1.00},
    panel       = {0.08, 0.10, 0.15, 1.00},
    panel_alt   = {0.11, 0.14, 0.21, 1.00},
    hover       = {0.16, 0.22, 0.34, 1.00},
    green       = {0.30, 0.95, 0.45, 1.00},
    red         = {1.00, 0.35, 0.35, 1.00},
}
UI.COLORS = {}
for k, v in pairs(UI.COLORS_DEFAULT) do UI.COLORS[k] = {v[1], v[2], v[3], v[4]} end

UI.state = {
    open = true,
    x = 100, y = 40, w = 820, h = 950,
    dragging = false, drag_ox = 0, drag_oy = 0,
    active_tab = 1,
    tabs = {},
    values = {}, colors = {}, keys = {},
    callbacks = {}, visible = {},
    open_combo = nil, open_combo_rect = nil,
    picker = nil, drag_target = nil,
    combo_just_opened = false,
    picker_just_opened = false,
    listening_key = nil, listen_wait_lmb_up = false,
    prev_lmb = false,
    toggle_key = 0x2D,
    scroll = 0,
    ui_settings = {
        accent      = {0.25, 0.60, 1.00, 1.00},
        opacity     = 96,
        border_glow = 100,
        window_w    = 820,
        window_h    = 950,
    },
}

function UI:apply_ui_settings()
    local us = self.state.ui_settings
    local base = self.COLORS_DEFAULT
    local acc = us.accent
    local op = (us.opacity or 96) / 100
    local bg = (us.border_glow or 100) / 100
    for k, v in pairs(base) do
        if type(v) == "table" then
            self.COLORS[k] = {v[1], v[2], v[3], (v[4] or 1) * op}
        end
    end
    self.COLORS.accent     = {acc[1], acc[2], acc[3], 1.00}
    self.COLORS.accent_dim = {acc[1], acc[2], acc[3], 0.25}
    self.COLORS.border      = {base.border[1]*bg, base.border[2]*bg, base.border[3]*bg, base.border[4]}
    self.COLORS.border_soft = {base.border_soft[1]*bg, base.border_soft[2]*bg, base.border_soft[3]*bg, base.border_soft[4]}
    self.state.w = us.window_w or 820
    self.state.h = us.window_h or 950
end

UI:apply_ui_settings()

local VK_NAMES = {
    [0x01] = "LMB", [0x02] = "RMB", [0x04] = "MMB",
    [0x08] = "Backspace", [0x09] = "Tab", [0x0D] = "Enter",
    [0x10] = "Shift", [0x11] = "Ctrl", [0x12] = "Alt",
    [0x13] = "Pause", [0x14] = "CapsLock", [0x1B] = "Escape",
    [0x20] = "Space", [0x21] = "PageUp", [0x22] = "PageDown",
    [0x23] = "End", [0x24] = "Home",
    [0x25] = "Left", [0x26] = "Up", [0x27] = "Right", [0x28] = "Down",
    [0x2C] = "PrintScr", [0x2D] = "Insert", [0x2E] = "Delete",
    [0x30] = "0", [0x31] = "1", [0x32] = "2", [0x33] = "3", [0x34] = "4",
    [0x35] = "5", [0x36] = "6", [0x37] = "7", [0x38] = "8", [0x39] = "9",
    [0x41] = "A", [0x42] = "B", [0x43] = "C", [0x44] = "D", [0x45] = "E",
    [0x46] = "F", [0x47] = "G", [0x48] = "H", [0x49] = "I", [0x4A] = "J",
    [0x4B] = "K", [0x4C] = "L", [0x4D] = "M", [0x4E] = "N", [0x4F] = "O",
    [0x50] = "P", [0x51] = "Q", [0x52] = "R", [0x53] = "S", [0x54] = "T",
    [0x55] = "U", [0x56] = "V", [0x57] = "W", [0x58] = "X", [0x59] = "Y",
    [0x5A] = "Z",
    [0x60] = "Num0", [0x61] = "Num1", [0x62] = "Num2", [0x63] = "Num3",
    [0x64] = "Num4", [0x65] = "Num5", [0x66] = "Num6", [0x67] = "Num7",
    [0x68] = "Num8", [0x69] = "Num9",
    [0x6A] = "Num*", [0x6B] = "Num+", [0x6D] = "Num-", [0x6E] = "Num.",
    [0x6F] = "Num/",
    [0x70] = "F1", [0x71] = "F2", [0x72] = "F3", [0x73] = "F4",
    [0x74] = "F5", [0x75] = "F6", [0x76] = "F7", [0x77] = "F8",
    [0x78] = "F9", [0x79] = "F10", [0x7A] = "F11", [0x7B] = "F12",
    [0x90] = "NumLock", [0x91] = "ScrollLock",
    [0xA0] = "LShift", [0xA1] = "RShift", [0xA2] = "LCtrl", [0xA3] = "RCtrl",
    [0xA4] = "LAlt", [0xA5] = "RAlt",
    [0xBA] = ";", [0xBB] = "=", [0xBC] = ",", [0xBD] = "-",
    [0xBE] = ".", [0xBF] = "/", [0xC0] = "`",
    [0xDB] = "[", [0xDC] = "\\", [0xDD] = "]", [0xDE] = "'",
}

local function vk_name(vk)
    if not vk or vk == 0 then return "[none]" end
    return "[" .. (VK_NAMES[vk] or ("0x" .. string.format("%02X", vk))) .. "]"
end

local sqrt = math.sqrt
local floor = math.floor
local abs = math.abs
local min = math.min
local max = math.max
local format = string.format

local function in_rect(mx, my, x, y, w, h)
    return mx >= x and mx <= x + w and my >= y and my <= y + h
end

local function mouse_pos()
    if utility and utility.get_mouse_pos then
        local ok, mx, my = pcall(utility.get_mouse_pos)
        if ok and mx and my then return mx, my end
    end
    if input and input.get_mouse_position then
        local ok, mx, my = pcall(input.get_mouse_position)
        if ok and mx and my then return mx, my end
    end
    return 0, 0
end

local function key_down(vk)
    if input and input.is_key_down then
        local ok, d = pcall(input.is_key_down, vk)
        return ok and d == true
    end
    return false
end

local function text_w(s, size)
    if draw and draw.get_text_size then
        local ok, w = pcall(draw.get_text_size, tostring(s or ""), size or 13)
        if ok and w then return w end
    end
    return #tostring(s or "") * (size or 13) * 0.55
end

function UI:add_tab(name, icon, size)
    self.state.tabs[#self.state.tabs + 1] = { name = name, icon = icon or "", groups = {} }
end

function UI:add_group(tab_name, group_name, width, same_line)
    for _, t in ipairs(self.state.tabs) do
        if t.name == tab_name then
            for _, g in ipairs(t.groups) do
                if g.name == group_name then return end
            end
            t.groups[#t.groups + 1] = { name = group_name, widgets = {} }
            return
        end
    end
    self:add_tab(tab_name)
    self:add_group(tab_name, group_name)
end

local function reg(self, tab, group, id, data)
    for _, t in ipairs(self.state.tabs) do
        if t.name == tab then
            for _, g in ipairs(t.groups) do
                if g.name == group then
                    data.id = id
                    g.widgets[#g.widgets + 1] = data
                    return
                end
            end
        end
    end
    self:add_group(tab, group)
    reg(self, tab, group, id, data)
end

function UI:add_checkbox(tab, group, id, label, default, opts)
    opts = opts or {}
    self.state.values[id] = default == true
    if opts.colorpicker then self.state.colors[id] = opts.colorpicker end
    if opts.key and opts.key ~= 0 then self.state.keys[id] = opts.key end
    reg(self, tab, group, id, { type="checkbox", label=label, default=default==true, color=opts.colorpicker, parent=opts.parent })
end

function UI:add_slider_int(tab, group, id, label, mn, mx, default, opts)
    opts = opts or {}
    self.state.values[id] = default or mn
    reg(self, tab, group, id, { type="slider_int", label=label, min=mn, max=mx, default=default, parent=opts.parent, fmt="%d" })
end

function UI:add_slider_float(tab, group, id, label, mn, mx, default, fmt, opts)
    opts = opts or {}
    self.state.values[id] = default or mn
    reg(self, tab, group, id, { type="slider_float", label=label, min=mn, max=mx, default=default, parent=opts.parent, fmt=fmt or "%.2f" })
end

function UI:add_combo(tab, group, id, label, items, default_idx, opts)
    opts = opts or {}
    self.state.values[id] = default_idx or 0
    reg(self, tab, group, id, { type="combo", label=label, items=items, default=default_idx or 0, parent=opts.parent })
end

function UI:add_multicombo(tab, group, id, label, items, defaults, opts)
    opts = opts or {}
    local def = {}
    for i = 1, #items do def[i] = defaults and defaults[i] == true end
    self.state.values[id] = def
    reg(self, tab, group, id, { type="multi", label=label, items=items, defaults=defaults, parent=opts.parent })
end

function UI:add_colorpicker(tab, group, id, label, default, opts)
    opts = opts or {}
    self.state.colors[id] = default or {1,1,1,1}
    reg(self, tab, group, id, { type="color", label=label, default=default, parent=opts.parent })
end

function UI:add_button(tab, group, id, label, callback)
    self.state.callbacks[id] = callback
    reg(self, tab, group, id, { type="button", label=label })
end

function UI:add_hotkey(tab, group, id, label, default_key, opts)
    opts = opts or {}
    self.state.keys[id] = default_key or 0
    reg(self, tab, group, id, { type="hotkey", label=label, default=default_key, parent=opts.parent })
end

function UI:add_separator(tab, group)
    reg(self, tab, group, "__sep_" .. tostring(math.random(999999)), { type="separator" })
end

function UI:add_label(tab, group, text)
    reg(self, tab, group, "__lbl_" .. tostring(math.random(999999)), { type="label", label=text })
end

function UI:get(id) return self.state.values[id] end
function UI:set(id, v)
    self.state.values[id] = v
    local cb = self.state.callbacks[id]
    if cb then pcall(cb, v) end
end
function UI:get_color(id) return self.state.colors[id] or {1,1,1,1} end
function UI:set_color(id, c) self.state.colors[id] = c end
function UI:get_key(id) return self.state.keys[id] or 0 end
function UI:set_key(id, k) self.state.keys[id] = k end
function UI:set_callback(id, cb) self.state.callbacks[id] = cb end
function UI:set_visible(id, v) self.state.visible[id] = v == true end

local function hsv_to_rgb(h, s, v)
    h = h * 6
    local i = floor(h)
    local f = h - i
    local p, q, t = v*(1-s), v*(1-f*s), v*(1-(1-f)*s)
    if i == 0 then return v, t, p end
    if i == 1 then return q, v, p end
    if i == 2 then return p, v, t end
    if i == 3 then return p, q, v end
    if i == 4 then return t, p, v end
    return v, p, q
end

local function rgb_to_hsv(r, g, b)
    local mx = max(r, g, b)
    local mn = min(r, g, b)
    local d = mx - mn
    local h = 0
    if d > 1e-6 then
        if mx == r then h = ((g - b) / d) % 6
        elseif mx == g then h = (b - r) / d + 2
        else h = (r - g) / d + 4 end
        h = h / 6
    end
    local s = (mx > 0) and (d / mx) or 0
    return h, s, mx
end

local picker_state = {
    hue = nil, sat = nil, val = nil,
    dragging_sv = false, dragging_hue = false,
}

local function draw_color_picker(self, id, x, y, w, h)
    local c = self:get_color(id)
    draw.rect_filled(x - 2, y - 2, w + 4, h + 4, {0,0,0,0.75}, 8)
    draw.rect(x, y, w, h, self.COLORS.border, 0, 1.5)

    local sq = min(w - 60, h - 60)
    local sx, sy = x + 12, y + 30
    local hue, sat, val = rgb_to_hsv(c[1], c[2], c[3])
    if picker_state.hue then
        hue, sat, val = picker_state.hue, picker_state.sat, picker_state.val
    end
    local steps = 12
    local cell = sq / steps
    for iy = 0, steps - 1 do
        for ix = 0, steps - 1 do
            local s = ix / (steps - 1)
            local v = 1 - iy / (steps - 1)
            local r, g, b = hsv_to_rgb(hue, s, v)
            draw.rect_filled(sx + ix*cell, sy + iy*cell, cell + 0.5, cell + 0.5, {r,g,b,1}, 0)
        end
    end
    draw.rect(sx, sy, sq, sq, self.COLORS.border, 0, 1)

    local hx, hy, hw, hh = sx + sq + 8, sy, 14, sq
    for i = 0, 17 do
        local t = i / 17
        local r, g, b = hsv_to_rgb(t, 1, 1)
        draw.rect_filled(hx, hy + i*(hh/18), hw, hh/18 + 0.5, {r,g,b,1}, 0)
    end
    draw.rect(hx, hy, hw, hh, self.COLORS.border, 0, 1)

    local sv_cursor_x = sx + sat * sq
    local sv_cursor_y = sy + (1 - val) * sq
    draw.circle(sv_cursor_x, sv_cursor_y, 6, {1,1,1,1}, 16, 2)
    draw.circle(sv_cursor_x, sv_cursor_y, 6, {0,0,0,1}, 16, 1)

    local hue_y = hy + hue * hh
    draw.rect_filled(hx - 2, hue_y - 2, hw + 4, 4, {1,1,1,1}, 2)
    draw.rect(hx - 2, hue_y - 2, hw + 4, 4, {0,0,0,1}, 2, 1)

    local px, py = x + 12, sy + sq + 8
    draw.rect_filled(px, py, sq, 20, c, 4)
    draw.rect(px, py, sq, 20, self.COLORS.border, 4, 1)

    local bx, by, bw, bh = x + w - 78, py, 66, 20
    draw.rect_filled(bx, by, bw, bh, self.COLORS.accent, 4)
    draw.text(bx + 22, by + 4, "OK", {1,1,1,1}, 12)

    local mx, my = mouse_pos()
    local lmb = key_down(0x01)

    if not lmb then
        picker_state.dragging_sv = false
        picker_state.dragging_hue = false
    end

    if self.state.picker_just_opened then return end

    if lmb then
        if picker_state.dragging_sv or in_rect(mx, my, sx, sy, sq, sq) then
            picker_state.dragging_sv = true
            sat = max(0, min(1, (mx - sx) / sq))
            val = max(0, min(1, 1 - (my - sy) / sq))
            local r, g, b = hsv_to_rgb(hue, sat, val)
            c[1], c[2], c[3] = r, g, b
            picker_state.hue, picker_state.sat, picker_state.val = hue, sat, val
            self:set_color(id, c)
            if self.state.callbacks[id] then pcall(self.state.callbacks[id], c) end
        elseif picker_state.dragging_hue or in_rect(mx, my, hx, hy, hw, hh) then
            picker_state.dragging_hue = true
            hue = max(0, min(1, (my - hy) / hh))
            local r, g, b = hsv_to_rgb(hue, sat, val)
            c[1], c[2], c[3] = r, g, b
            picker_state.hue, picker_state.sat, picker_state.val = hue, sat, val
            self:set_color(id, c)
            if self.state.callbacks[id] then pcall(self.state.callbacks[id], c) end
        elseif in_rect(mx, my, bx, by, bw, bh) then
            self.state.picker = nil
            picker_state.hue = nil
        end
    end
end

local function draw_widget(self, w, x, y, wd, h, lmb_down, lmb_click, block_input)
    if w.type == "separator" then
        draw.line(x + 4, y + h/2, x + wd - 4, y + h/2, self.COLORS.border_soft, 1)
        return
    elseif w.type == "label" then
        draw.text(x + 4, y + (h - 12)/2, w.label, self.COLORS.text_muted, 11)
        return
    end

    local mx, my = mouse_pos()
    local hovered = in_rect(mx, my, x, y, wd, h) and not block_input

    if w.type == "checkbox" then
        local on = self.state.values[w.id] == true
        if hovered then draw.rect_filled(x, y, wd, h, self.COLORS.hover, 4) end
        local sw, sh = 30, 16
        local sx, sy = x + 4, y + (h - sh) / 2
        local track = on and self.COLORS.accent or self.COLORS.border_soft
        draw.rect_filled(sx, sy, sw, sh, track, sh/2)
        local knob = sh - 4
        local kx = on and (sx + sw - knob - 2) or (sx + 2)
        draw.circle_filled(kx + knob/2, sy + sh/2, knob/2, self.COLORS.text, 16)
        draw.text(sx + sw + 8, y + (h - 13)/2, w.label,
            on and self.COLORS.text or self.COLORS.text_dim, 13)
        if self.state.colors[w.id] ~= nil then
            local c = self:get_color(w.id)
            local cx = x + wd - 20
            local cy = y + (h - 14) / 2
            draw.rect_filled(cx, cy, 14, 14, c, 4)
            draw.rect(cx, cy, 14, 14, self.COLORS.border, 4, 1)
            if lmb_click and hovered and in_rect(mx, my, cx - 3, cy - 3, 20, 20) then
                self.state.picker = { id = w.id, x = cx - 100, y = y + h + 4, w = 200, h = 200 }
                self.state.picker_just_opened = true
                picker_state.hue = nil
            end
        end
        if lmb_click and hovered and not (self.state.colors[w.id] and in_rect(mx, my, x + wd - 20, y, 20, h)) then
            self.state.values[w.id] = not on
            if self.state.callbacks[w.id] then pcall(self.state.callbacks[w.id], self.state.values[w.id]) end
        end

    elseif w.type == "slider_int" or w.type == "slider_float" then
        local v = tonumber(self.state.values[w.id]) or w.default or w.min
        local slider_y = y + h - 9
        local slider_h = 5
        local slider_w = wd - 8
        local track_x = x + 4
        local fmt = w.fmt or "%d"
        local vtxt = format(fmt, v)
        local vw = text_w(vtxt, 12)
        draw.text(x + 4, y + 2, w.label, self.COLORS.text, 12)
        draw.text(x + wd - vw - 4, y + 2, vtxt, self.COLORS.accent, 12)
        draw.rect_filled(track_x, slider_y, slider_w, slider_h, self.COLORS.border_soft, slider_h/2)
        local t = (w.max > w.min) and ((v - w.min) / (w.max - w.min)) or 0
        draw.rect_filled(track_x, slider_y, slider_w * t, slider_h, self.COLORS.accent, slider_h/2)
        draw.circle_filled(track_x + slider_w * t, slider_y + slider_h/2, 6, self.COLORS.text, 16)
        local hot = in_rect(mx, my, track_x, slider_y - 6, slider_w, slider_h + 12) and not block_input
        if lmb_down then
            if not self.state.drag_target and hot then self.state.drag_target = w.id end
            if self.state.drag_target == w.id then
                local nt = max(0, min(1, (mx - track_x) / slider_w))
                local nv = w.min + (w.max - w.min) * nt
                if w.type == "slider_int" then nv = floor(nv + 0.5) end
                self.state.values[w.id] = nv
                if self.state.callbacks[w.id] then pcall(self.state.callbacks[w.id], nv) end
            end
        else
            if self.state.drag_target == w.id then self.state.drag_target = nil end
        end

    elseif w.type == "combo" then
        local idx = tonumber(self.state.values[w.id]) or 0
        if hovered then draw.rect_filled(x, y, wd, h, self.COLORS.hover, 4) end
        draw.text(x + 6, y + (h - 13)/2, w.label, self.COLORS.text_dim, 12)
        local cur = w.items[idx + 1] or "-"
        local cw = text_w(cur, 12)
        draw.text(x + wd - cw - 16, y + (h - 13)/2, cur, self.COLORS.text, 12)
        draw.text(x + wd - 10, y + (h - 13)/2, "v", self.COLORS.text_dim, 10)
        if lmb_click and hovered then
            if self.state.open_combo == w.id then
                self.state.open_combo = nil
                self.state.open_combo_rect = nil
            else
                self.state.open_combo = w.id
                self.state.combo_just_opened = true
                self.state.open_combo_rect = { x = x, y = y + h, w = wd, widget = w }
            end
        end

    elseif w.type == "multi" then
        local vals = self.state.values[w.id] or {}
        if hovered then draw.rect_filled(x, y, wd, h, self.COLORS.hover, 4) end
        draw.text(x + 6, y + (h - 13)/2, w.label, self.COLORS.text_dim, 12)
        local n = 0
        for i = 1, #w.items do if vals[i] then n = n + 1 end end
        local txt = (n == 0) and "None" or (n .. " selected")
        local tw = text_w(txt, 12)
        draw.text(x + wd - tw - 8, y + (h - 13)/2, txt, self.COLORS.accent, 12)
        if lmb_click and hovered then
            if self.state.open_combo == w.id then
                self.state.open_combo = nil
                self.state.open_combo_rect = nil
            else
                self.state.open_combo = w.id
                self.state.combo_just_opened = true
                self.state.open_combo_rect = { x = x, y = y + h, w = wd, widget = w }
            end
        end

    elseif w.type == "button" then
        local bg = hovered and self.COLORS.hover or self.COLORS.panel_alt
        draw.rect_filled(x, y + 2, wd, h - 4, bg, 4)
        draw.rect(x, y + 2, wd, h - 4, self.COLORS.border_soft, 4, 1)
        local tw = text_w(w.label, 12)
        draw.text(x + (wd - tw)/2, y + (h - 13)/2, w.label, self.COLORS.text, 12)
        if lmb_click and hovered then
            if self.state.callbacks[w.id] then pcall(self.state.callbacks[w.id]) end
        end

    elseif w.type == "hotkey" then
        local k = self.state.keys[w.id] or 0
        if hovered then draw.rect_filled(x, y, wd, h, self.COLORS.hover, 4) end
        draw.text(x + 6, y + (h - 13)/2, w.label, self.COLORS.text, 12)
        local kname
        if self.state.listening_key == w.id then kname = "[press...]"
        else kname = vk_name(k) end
        local kw = text_w(kname, 12)
        draw.rect_filled(x + wd - kw - 14, y + 4, kw + 10, h - 8, self.COLORS.panel_alt, 4)
        draw.rect(x + wd - kw - 14, y + 4, kw + 10, h - 8, self.COLORS.border_soft, 4, 1)
        draw.text(x + wd - kw - 9, y + (h - 13)/2, kname, self.COLORS.accent, 12)
        if lmb_click and hovered then
            self.state.listening_key = w.id
            self.state.listen_wait_lmb_up = true
        end

    elseif w.type == "color" then
        local c = self:get_color(w.id)
        if hovered then draw.rect_filled(x, y, wd, h, self.COLORS.hover, 4) end
        draw.text(x + 6, y + (h - 13)/2, w.label, self.COLORS.text, 12)
        local cx = x + wd - 24
        draw.rect_filled(cx, y + 4, 16, h - 8, c, 3)
        draw.rect(cx, y + 4, 16, h - 8, self.COLORS.border, 3, 1)
        if lmb_click and in_rect(mx, my, cx - 3, y, 22, h) then
            self.state.picker = { id = w.id, x = cx - 100, y = y + h + 4, w = 200, h = 200 }
            self.state.picker_just_opened = true
            picker_state.hue = nil
        end
    end
end

local function draw_open_combo_overlay(self)
    if not self.state.open_combo then return end
    local r = self.state.open_combo_rect
    if not r then return end
    local w = r.widget
    if not w then return end

    local mx, my = mouse_pos()
    local lmb_down = key_down(0x01)
    local lmb_click = lmb_down and not self.state.prev_lmb

    local x, y, wd = r.x, r.y, r.w
    local item_h = 20
    local list_h = #w.items * item_h

    if w.type == "combo" then
        local idx = tonumber(self.state.values[w.id]) or 0
        draw.rect_filled(x, y, wd, list_h, self.COLORS.panel_alt, 4)
        draw.rect(x, y, wd, list_h, self.COLORS.border, 4, 1)
        for i, item in ipairs(w.items) do
            local iy = y + (i - 1) * item_h
            local ih = in_rect(mx, my, x, iy, wd, item_h)
            if ih then draw.rect_filled(x + 2, iy + 1, wd - 4, item_h - 2, self.COLORS.hover, 2) end
            local c = (i - 1 == idx) and self.COLORS.accent or self.COLORS.text
            draw.text(x + 8, iy + 4, item, c, 12)
            if lmb_click and ih and not self.state.combo_just_opened then
                self.state.values[w.id] = i - 1
                self.state.open_combo = nil
                self.state.open_combo_rect = nil
                if self.state.callbacks[w.id] then pcall(self.state.callbacks[w.id], i - 1) end
            end
        end

    elseif w.type == "multi" then
        local vals = self.state.values[w.id] or {}
        draw.rect_filled(x, y, wd, list_h, self.COLORS.panel_alt, 4)
        draw.rect(x, y, wd, list_h, self.COLORS.border, 4, 1)
        for i, item in ipairs(w.items) do
            local iy = y + (i - 1) * item_h
            local ih = in_rect(mx, my, x, iy, wd, item_h)
            if ih then draw.rect_filled(x + 2, iy + 1, wd - 4, item_h - 2, self.COLORS.hover, 2) end
            draw.rect_filled(x + 6, iy + 5, 10, 10, self.COLORS.border_soft, 2)
            if vals[i] then draw.rect_filled(x + 8, iy + 7, 6, 6, self.COLORS.accent, 1) end
            draw.text(x + 22, iy + 4, item, self.COLORS.text, 12)
            if lmb_click and ih and not self.state.combo_just_opened then
                vals[i] = not vals[i]
                self.state.values[w.id] = vals
                if self.state.callbacks[w.id] then pcall(self.state.callbacks[w.id], vals) end
            end
        end
    end

    if lmb_click and not self.state.combo_just_opened then
        if not in_rect(mx, my, x, y, wd, list_h) then
            self.state.open_combo = nil
            self.state.open_combo_rect = nil
        end
    end
end

local function draw_window(self)
    local st = self.state
    if not st.open then
        st.prev_lmb = key_down(0x01)
        return
    end

    local mx, my = mouse_pos()
    local lmb_down = key_down(0x01)
    local lmb_click = lmb_down and not st.prev_lmb

    draw.rect_filled(st.x + 4, st.y + 4, st.w, st.h, {0,0,0,0.35}, 6)
    draw.rect_filled(st.x, st.y, st.w, st.h, self.COLORS.bg, 6)
    draw.rect(st.x, st.y, st.w, st.h, self.COLORS.border, 6, 1.5)

    local th = 34
    draw.rect_filled(st.x + 1, st.y + 1, st.w - 2, th, self.COLORS.titlebar, 5)
    draw.line(st.x, st.y + th, st.x + st.w, st.y + th, self.COLORS.border_soft, 1)
    draw.text(st.x + 14, st.y + 9, "DELTA V2", self.COLORS.accent, 16)
    local vtxt = "v" .. VERSION
    local vw = text_w(vtxt, 11)
    draw.text(st.x + st.w - vw - 14, st.y + 11, vtxt, self.COLORS.text_muted, 11)

    if lmb_down then
        if not st.dragging and in_rect(mx, my, st.x, st.y, st.w, th) then
            st.dragging = true
            st.drag_ox = mx - st.x
            st.drag_oy = my - st.y
        elseif st.dragging then
            st.x = mx - st.drag_ox
            st.y = my - st.drag_oy
        end
    else
        st.dragging = false
    end

    local tab_y = st.y + th
    local tab_h = 34
    draw.rect_filled(st.x + 1, tab_y, st.w - 2, tab_h, self.COLORS.panel, 0)
    local tx = st.x + 8
    for i, tab in ipairs(st.tabs) do
        local tw = text_w(tab.name, 13) + 32
        local active = (i == st.active_tab)
        local hovered = in_rect(mx, my, tx, tab_y + 4, tw, tab_h - 8)
        if active then
            draw.rect_filled(tx, tab_y + 4, tw, tab_h - 8, self.COLORS.accent_dim, 4)
            draw.rect_filled(tx, tab_y + tab_h - 3, tw, 2, self.COLORS.accent, 0)
        elseif hovered then
            draw.rect_filled(tx, tab_y + 4, tw, tab_h - 8, self.COLORS.hover, 4)
        end
        local c = active and self.COLORS.text or self.COLORS.text_dim
        draw.text(tx + 16, tab_y + 12, tab.name, c, 13)
        if lmb_click and hovered and not st.combo_just_opened and not st.picker_just_opened then
            st.active_tab = i
            st.scroll = 0
            st.open_combo = nil
            st.open_combo_rect = nil
        end
        tx = tx + tw + 4
    end
    draw.line(st.x, tab_y + tab_h, st.x + st.w, tab_y + tab_h, self.COLORS.border_soft, 1)

    local body_y = tab_y + tab_h + 4
    local body_bottom = st.y + st.h - 8
    local cur_tab = st.tabs[st.active_tab]
    if not cur_tab then
        st.prev_lmb = lmb_down
        return
    end

    local block_input = (st.open_combo ~= nil) or (st.picker ~= nil)

    local pad = 8
    local col_w = floor((st.w - pad * 3) / 2)
    local cols = { st.x + pad, st.x + pad * 2 + col_w }

    local group_heights = {}
    local col_h = { 0, 0 }
    local col_assign = {}
    for i, g in ipairs(cur_tab.groups) do
        local hh = 26 + 10
        for _, w in ipairs(g.widgets) do
            if self.state.visible[w.id] ~= false then hh = hh + 26 + 4 end
        end
        group_heights[i] = hh
        local ci = (col_h[1] <= col_h[2]) and 1 or 2
        col_assign[i] = ci
        col_h[ci] = col_h[ci] + hh + 10
    end
    local total_h = max(col_h[1], col_h[2]) + pad * 2
    local view_h = body_bottom - body_y
    local max_scroll = max(0, total_h - view_h)
    st.scroll = st.scroll or 0
    st.scroll = max(0, min(max_scroll, st.scroll))

    local ys = { body_y + pad - st.scroll, body_y + pad - st.scroll }

    for i, g in ipairs(cur_tab.groups) do
        local ci = col_assign[i]
        local gx = cols[ci]
        local gy = ys[ci]

        local header_h = 26
        local row_h = 26
        local content_h = 0
        for _, w in ipairs(g.widgets) do
            if self.state.visible[w.id] ~= false then
                content_h = content_h + row_h + 4
            end
        end
        local group_h = header_h + content_h + 10

        if gy < body_bottom and gy + group_h > body_y then
            local clip_top = max(gy, body_y)
            local clip_bottom = min(gy + group_h, body_bottom)
            local clip_h = clip_bottom - clip_top
            if clip_h > 0 then
                draw.rect_filled(gx, clip_top, col_w, clip_h, self.COLORS.panel, 5)
                draw.rect(gx, clip_top, col_w, clip_h, self.COLORS.border_soft, 5, 1)

                if gy >= body_y and gy < body_bottom then
                    draw.rect_filled(gx + 1, gy + 1, col_w - 2, header_h - 2, self.COLORS.panel_alt, 4)
                    draw.rect_filled(gx + 1, gy + 1, 3, header_h - 2, self.COLORS.accent, 2)
                    draw.text(gx + 12, gy + 7, g.name, self.COLORS.text, 13)
                end

                local wy = gy + header_h + 2
                for _, w in ipairs(g.widgets) do
                    if self.state.visible[w.id] ~= false then
                        if wy >= body_y and wy + row_h <= body_bottom then
                            draw_widget(self, w, gx + 6, wy, col_w - 12, row_h, lmb_down, lmb_click, block_input)
                        end
                        wy = wy + row_h + 4
                    end
                end
            end
        end
        ys[ci] = gy + group_h + 10
    end

    if max_scroll > 0 then
        local sb_x = st.x + st.w - 6
        local sb_w = 4
        local bar_h = max(20, view_h * (view_h / total_h))
        local bar_y = body_y + (view_h - bar_h) * (st.scroll / max_scroll)
        draw.rect_filled(sb_x, body_y, sb_w, view_h, {1,1,1,0.05}, 2)
        draw.rect_filled(sb_x, bar_y, sb_w, bar_h, self.COLORS.accent, 2)
    end

    draw_open_combo_overlay(self)

    if st.picker then
        local p = st.picker
        draw.rect_filled(p.x - 2, p.y - 2, p.w + 4, p.h + 4, {0,0,0,0.6}, 8)
        draw_color_picker(self, p.id, p.x, p.y, p.w, p.h)
        if lmb_click and not st.picker_just_opened and not picker_state.dragging_sv and not picker_state.dragging_hue then
            if not in_rect(mx, my, p.x - 10, p.y - 10, p.w + 20, p.h + 20) then
                st.picker = nil
                picker_state.hue = nil
            end
        end
    end

    st.combo_just_opened = false
    st.picker_just_opened = false
    st.prev_lmb = lmb_down
end

local function process_hotkey_listening(self)
    if not self.state.listening_key then return end
    if self.state.listen_wait_lmb_up then
        if not key_down(0x01) then self.state.listen_wait_lmb_up = false end
        return
    end
    if key_down(0x1B) then
        self.state.listening_key = nil
        return
    end
    for vk = 1, 254 do
        if vk ~= 0x01 and key_down(vk) then
            self.state.keys[self.state.listening_key] = vk
            if self.state.listening_key == "v4_ui_toggle_key" then
                self.state.toggle_key = vk
                _prev_toggle = true
            end
            self.state.listening_key = nil
            break
        end
    end
end

local _prev_toggle = false
local function set_cursor_visible(visible)
    pcall(function()
        local sg = game:GetService("StarterGui")
        if sg.SetCore then sg:SetCore("MouseIconEnabled", visible) end
    end)
    pcall(function()
        local uis = game:GetService("UserInputService")
        if uis then
            uis.MouseIconEnabled = visible
            if uis.MouseBehavior ~= nil then
                uis.MouseBehavior = visible and Enum.MouseBehavior.Default or Enum.MouseBehavior.LockCenter
            end
        end
    end)
end

local function process_toggle(self)
    local toggle_key = self.state.toggle_key
    if not toggle_key or toggle_key == 0 then
        _prev_toggle = false
        return
    end
    if self.state.listening_key == "v4_ui_toggle_key" then
        _prev_toggle = key_down(toggle_key)
        return
    end
    local kd = key_down(toggle_key)
    if kd and not _prev_toggle then
        self.state.open = not self.state.open
        set_cursor_visible(self.state.open)
    end
    _prev_toggle = kd
end

local base_ui = setmetatable({}, UI)

local _menu = menu
menu = setmetatable({}, {
    __index = function(_, k)
        if base_ui[k] then return function(_, ...) return base_ui[k](base_ui, ...) end end
        if _menu and _menu[k] then return _menu[k] end
        return nil
    end,
})

local function bind(name, fn)
    menu[name] = function(...) return fn(...) end
end

bind("add_tab", function(t, i, s) base_ui:add_tab(t, i, s) end)
bind("AddTab", menu.add_tab)
bind("add_group", function(t, g, w, s) base_ui:add_group(t, g, w, s) end)
bind("AddGroup", menu.add_group)
bind("add_checkbox", function(t, g, id, l, d, o) base_ui:add_checkbox(t, g, id, l, d, o) end)
bind("AddCheckbox", menu.add_checkbox)
bind("add_slider_int", function(...) base_ui:add_slider_int(...) end)
bind("AddSliderInt", menu.add_slider_int)
bind("add_slider_float", function(...) base_ui:add_slider_float(...) end)
bind("AddSliderFloat", menu.add_slider_float)
bind("add_combo", function(...) base_ui:add_combo(...) end)
bind("AddCombo", menu.add_combo)
bind("add_multicombo", function(...) base_ui:add_multicombo(...) end)
bind("AddMulticombo", menu.add_multicombo)
bind("add_colorpicker", function(...) base_ui:add_colorpicker(...) end)
bind("AddColorpicker", menu.add_colorpicker)
bind("add_button", function(...) base_ui:add_button(...) end)
bind("AddButton", menu.add_button)
bind("add_hotkey", function(...) base_ui:add_hotkey(...) end)
bind("AddHotkey", menu.add_hotkey)
bind("add_separator", function(...) base_ui:add_separator(...) end)
bind("AddSeparator", menu.add_separator)
bind("add_label", function(...) base_ui:add_label(...) end)
bind("AddLabel", menu.add_label)
bind("get", function(id) return base_ui:get(id) end)
bind("Get", menu.get)
bind("set", function(id, v) base_ui:set(id, v) end)
bind("Set", menu.set)
bind("get_color", function(id) return base_ui:get_color(id) end)
bind("GetColor", menu.get_color)
bind("set_color", function(id, c) base_ui:set_color(id, c) end)
bind("SetColor", menu.set_color)
bind("get_key", function(id) return base_ui:get_key(id) end)
bind("GetKey", menu.get_key)
bind("set_key", function(id, k) base_ui:set_key(id, k) end)
bind("SetKey", menu.set_key)
bind("set_callback", function(id, cb) base_ui:set_callback(id, cb) end)
bind("SetCallback", menu.set_callback)
bind("set_visible", function(id, v) base_ui:set_visible(id, v) end)
bind("SetVisible", menu.set_visible)

local function tick_and_render()
    process_hotkey_listening(base_ui)
    process_toggle(base_ui)
    draw_window(base_ui)
end

-- =====================================================================
-- PARTE 2 — SCRIPT DELTA V2 (registros e features)
-- =====================================================================

menu.add_tab("Aimbot",  "A")
menu.add_tab("Visuals", "V")
menu.add_tab("World",   "W")
menu.add_tab("Misc",    "M")
menu.add_tab("Config",  "C")

-- ============ VISUALS — PLAYER ESP ============
menu.add_group("Visuals", "ESP — Player")
menu.add_checkbox("Visuals", "ESP — Player", "v4_player_enabled", "=== PLAYER === Enable Player ESP", false)
menu.add_checkbox("Visuals", "ESP — Player", "v4_player_box", "Box", true, { parent = "v4_player_enabled", colorpicker = {1, 0.3, 0.3, 1} })
menu.add_checkbox("Visuals", "ESP — Player", "v4_player_health", "Health Bar", true, { parent = "v4_player_enabled" })
menu.add_checkbox("Visuals", "ESP — Player", "v4_player_name", "Name", true, { parent = "v4_player_enabled", colorpicker = {1, 1, 1, 1} })
menu.add_checkbox("Visuals", "ESP — Player", "v4_player_dist", "Distance", true, { parent = "v4_player_enabled", colorpicker = {0.7, 0.7, 0.7, 1} })
menu.add_checkbox("Visuals", "ESP — Player", "v4_player_skeleton", "Skeleton", false, { parent = "v4_player_enabled", colorpicker = {1, 1, 1, 1} })
menu.add_checkbox("Visuals", "ESP — Player", "v4_player_weapon", "Weapon Name", true, { parent = "v4_player_enabled", colorpicker = {1, 0.8, 0.2, 1} })
menu.add_checkbox("Visuals", "ESP — Player", "v4_player_team_check", "Team Check", false, { parent = "v4_player_enabled" })
menu.add_slider_int("Visuals", "ESP — Player", "v4_player_range", "Player Range (m)", 10, 1500, 500, { parent = "v4_player_enabled" })

-- ============ VISUALS — NPC ESP ============
menu.add_group("Visuals", "ESP — NPC")
menu.add_checkbox("Visuals", "ESP — NPC", "v4_npc_enabled", "=== NPC === Enable NPC ESP", false)
menu.add_checkbox("Visuals", "ESP — NPC", "v4_npc_box", "Box", true, { parent = "v4_npc_enabled", colorpicker = {1, 0.3, 0.3, 1} })
menu.add_checkbox("Visuals", "ESP — NPC", "v4_npc_health", "Health Bar", true, { parent = "v4_npc_enabled" })
menu.add_checkbox("Visuals", "ESP — NPC", "v4_npc_name", "Name", true, { parent = "v4_npc_enabled", colorpicker = {1, 0.3, 0.3, 1} })
menu.add_checkbox("Visuals", "ESP — NPC", "v4_npc_dist", "Distance", true, { parent = "v4_npc_enabled", colorpicker = {0.7, 0.7, 0.7, 1} })
menu.add_checkbox("Visuals", "ESP — NPC", "v4_npc_skeleton", "Skeleton", false, { parent = "v4_npc_enabled", colorpicker = {1, 1, 1, 1} })
menu.add_slider_int("Visuals", "ESP — NPC", "v4_npc_range", "NPC Range (m)", 10, 600, 140, { parent = "v4_npc_enabled" })

-- ============ WORLD — CAR ESP ============
menu.add_group("World", "ESP — Car")
menu.add_checkbox("World", "ESP — Car", "v4_car_enabled", "=== CAR === Enable Car ESP", false)
menu.add_checkbox("World", "ESP — Car", "v4_car_col", "Color", true, { parent = "v4_car_enabled", colorpicker = {0.4, 1, 0.4, 1} })
menu.add_slider_int("World", "ESP — Car", "v4_car_range", "Car Range (m)", 50, 2000, 500, { parent = "v4_car_enabled" })

-- ============ WORLD — EXTRACT ============
menu.add_group("World", "ESP — Extract")
menu.add_checkbox("World", "ESP — Extract", "v4_exit_enabled", "=== EXTRACT === Enable Extract ESP", false)
menu.add_checkbox("World", "ESP — Extract", "v4_exit_col", "Color", true, { parent = "v4_exit_enabled", colorpicker = {1, 0.9, 0.2, 1} })
menu.add_slider_int("World", "ESP — Extract", "v4_exit_range", "Extract Range (m)", 50, 2000, 300, { parent = "v4_exit_enabled" })

-- ============ VISUALS — LOOT ESP ============
menu.add_group("Visuals", "ESP — Loot")
menu.add_checkbox("Visuals", "ESP — Loot", "v4_loot_enabled", "=== LOOT === Enable Loot ESP", false)
menu.add_checkbox("Visuals", "ESP — Loot", "v4_loot_weapons", "Weapons", true, { parent = "v4_loot_enabled" })
menu.add_checkbox("Visuals", "ESP — Loot", "v4_loot_armor", "Armor", true, { parent = "v4_loot_enabled" })
menu.add_checkbox("Visuals", "ESP — Loot", "v4_loot_valuables", "Valuables", true, { parent = "v4_loot_enabled" })
menu.add_checkbox("Visuals", "ESP — Loot", "v4_loot_meds", "Meds", false, { parent = "v4_loot_enabled" })
menu.add_checkbox("Visuals", "ESP — Loot", "v4_loot_ammo", "Ammo", false, { parent = "v4_loot_enabled" })
menu.add_checkbox("Visuals", "ESP — Loot", "v4_loot_other", "Other", false, { parent = "v4_loot_enabled" })
menu.add_checkbox("Visuals", "ESP — Loot", "v4_loot_prefix", "Category Prefix", true, { parent = "v4_loot_enabled" })
menu.add_slider_int("Visuals", "ESP — Loot", "v4_loot_range", "Loot Range (m)", 10, 600, 84, { parent = "v4_loot_enabled" })

-- ============ WORLD — CORPSE ============
menu.add_group("World", "ESP — Corpse")
menu.add_checkbox("World", "ESP — Corpse", "v4_corpse_enabled", "=== CORPSE === Enable Corpse ESP", false)
menu.add_checkbox("World", "ESP — Corpse", "v4_corpse_name", "Name", true, { parent = "v4_corpse_enabled", colorpicker = {1, 0.3, 0.3, 1} })
menu.add_checkbox("World", "ESP — Corpse", "v4_corpse_dist", "Distance", true, { parent = "v4_corpse_enabled", colorpicker = {0.7, 0.7, 0.7, 1} })
menu.add_checkbox("World", "ESP — Corpse", "v4_corpse_marker", "Marker (X)", true, { parent = "v4_corpse_enabled", colorpicker = {1, 0, 0, 1} })
menu.add_slider_int("World", "ESP — Corpse", "v4_corpse_range", "Corpse Range (m)", 10, 600, 200, { parent = "v4_corpse_enabled" })

-- ============ WORLD — CONTAINER ============
menu.add_group("World", "ESP — Container")
menu.add_checkbox("World", "ESP — Container", "v4_container_enabled", "=== CONTAINER === Enable Container ESP", false)
menu.add_checkbox("World", "ESP — Container", "v4_container_col", "Color", true, { parent = "v4_container_enabled", colorpicker = {0.5, 0.5, 1, 1} })
menu.add_slider_int("World", "ESP — Container", "v4_container_range", "Container Range (m)", 10, 600, 84, { parent = "v4_container_enabled" })
menu.add_checkbox("World", "ESP — Container", "v4_container_contents", "Show Contents", false, { parent = "v4_container_enabled" })

-- ============ WORLD — QUEST ============
menu.add_group("World", "ESP — Quest")
menu.add_checkbox("World", "ESP — Quest", "v4_quest_enabled", "=== QUEST === Enable Quest ESP", false)
menu.add_checkbox("World", "ESP — Quest", "v4_quest_col", "Color", true, { parent = "v4_quest_enabled", colorpicker = {1, 0.5, 1, 1} })
menu.add_slider_int("World", "ESP — Quest", "v4_quest_range", "Quest Range (m)", 10, 1500, 140, { parent = "v4_quest_enabled" })

-- ============ WORLD — CLAYMORE ============
menu.add_group("World", "ESP — Claymore")
menu.add_checkbox("World", "ESP — Claymore", "v4_claymore_enabled", "=== CLAYMORE === Enable Claymore ESP", false)
menu.add_checkbox("World", "ESP — Claymore", "v4_claymore_col", "Color", true, { parent = "v4_claymore_enabled", colorpicker = {1, 0.3, 0.1, 1} })
menu.add_slider_int("World", "ESP — Claymore", "v4_claymore_range", "Claymore Range (m)", 10, 600, 140, { parent = "v4_claymore_enabled" })
menu.add_checkbox("World", "ESP — Claymore", "v4_claymore_names", "Show Name + Distance", false, { parent = "v4_claymore_enabled" })

-- ============ VISUALS — CHAMS ============
menu.add_group("Visuals", "Chams")
menu.add_checkbox("Visuals", "Chams", "v4_chams_enabled", "=== CHAMS === Enable Chams", false)
menu.add_combo("Visuals", "Chams", "v4_chams_style", "Style", { "Filled", "Outline", "Glow" }, 0, { parent = "v4_chams_enabled" })
menu.add_checkbox("Visuals", "Chams", "v4_chams_players", "Players", true, { parent = "v4_chams_enabled" })
menu.add_checkbox("Visuals", "Chams", "v4_chams_npcs", "NPCs", true, { parent = "v4_chams_enabled" })
menu.add_colorpicker("Visuals", "Chams", "v4_chams_color", "Visible Color", {0, 1, 0, 1})
menu.add_checkbox("Visuals", "Chams", "v4_chams_gradient", "Visibility Check (Gradient)", false, { parent = "v4_chams_enabled" })
menu.add_colorpicker("Visuals", "Chams", "v4_chams_color2", "Hidden Color", {1, 0, 0, 1})
menu.add_slider_int("Visuals", "Chams", "v4_chams_vis_fov", "Visibility FOV", 0, 360, 90, { parent = "v4_chams_gradient" })

-- ============ WORLD — BOSS TRACKER ============
menu.add_group("World", "Boss Tracker")
menu.add_checkbox("World", "Boss Tracker", "v4_boss_enabled", "Enable Boss Tracker", false)

-- ============ VISUALS — LOOT TRACKER ============
menu.add_group("Visuals", "Loot Tracker")
menu.add_checkbox("Visuals", "Loot Tracker", "v4_lt_enabled", "Enable Loot Tracker", false)
menu.add_slider_int("Visuals", "Loot Tracker", "v4_lt_max", "Max Entries", 1, 50, 50, { parent = "v4_lt_enabled" })
menu.add_slider_int("Visuals", "Loot Tracker", "v4_lt_far", "Too Far Threshold (m)", 100, 5000, 1000, { parent = "v4_lt_enabled" })

-- ============ AIMBOT ============
menu.add_group("Aimbot", "Aimbot")
menu.add_checkbox("Aimbot", "Aimbot", "v4_aim_enabled", "Enable Aimbot", false)
menu.add_hotkey("Aimbot", "Aimbot", "v4_aim_keybind", "Aim Key", 0x02, { parent = "v4_aim_enabled" })
menu.add_combo("Aimbot", "Aimbot", "v4_aim_target", "Target", { "Players + NPCs", "Players Only", "NPCs Only" }, 0, { parent = "v4_aim_enabled" })
menu.add_combo("Aimbot", "Aimbot", "v4_aim_bone", "Bone", { "Head", "UpperTorso", "LowerTorso" }, 0, { parent = "v4_aim_enabled" })
menu.add_slider_int("Aimbot", "Aimbot", "v4_aim_fov", "FOV", 10, 800, 150, { parent = "v4_aim_enabled" })
menu.add_slider_int("Aimbot", "Aimbot", "v4_aim_smooth", "Smooth", 1, 20, 4, { parent = "v4_aim_enabled" })
menu.add_checkbox("Aimbot", "Aimbot", "v4_aim_predict", "Ballistic Prediction", true, { parent = "v4_aim_enabled" })
menu.add_slider_int("Aimbot", "Aimbot", "v4_aim_predict_scale", "Predict Scale %", 0, 200, 120, { parent = "v4_aim_enabled" })
menu.add_checkbox("Aimbot", "Aimbot", "v4_aim_lead", "Velocity Lead", true, { parent = "v4_aim_enabled" })
menu.add_checkbox("Aimbot", "Aimbot", "v4_aim_visible", "Visible Only", true, { parent = "v4_aim_enabled" })
menu.add_checkbox("Aimbot", "Aimbot", "v4_aim_draw_fov", "Draw FOV Circle", true, { parent = "v4_aim_enabled" })
menu.add_checkbox("Aimbot", "Aimbot", "v4_aim_lock", "Target Lock (Sticky)", false, { parent = "v4_aim_enabled" })

-- ============ AIMBOT — INVENTORY CHECKER ============
menu.add_group("Aimbot", "Inventory Checker")
menu.add_checkbox("Aimbot", "Inventory Checker", "v4_inv_enabled", "Enable Inventory Checker", false)
menu.add_hotkey("Aimbot", "Inventory Checker", "v4_inv_keybind", "Toggle Key", 0x02, { parent = "v4_inv_enabled" })
menu.add_combo("Aimbot", "Inventory Checker", "v4_inv_mode", "Show Mode", { "Always", "Toggle", "Hold" }, 0, { parent = "v4_inv_enabled" })
menu.add_checkbox("Aimbot", "Inventory Checker", "v4_inv_guns", "Show Guns", true, { parent = "v4_inv_enabled" })
menu.add_slider_int("Aimbot", "Inventory Checker", "v4_inv_max_guns", "Max Guns", 1, 20, 10, { parent = "v4_inv_enabled" })
menu.add_checkbox("Aimbot", "Inventory Checker", "v4_inv_armor", "Show Armor", true, { parent = "v4_inv_enabled" })
menu.add_slider_int("Aimbot", "Inventory Checker", "v4_inv_max_armor", "Max Armor", 1, 20, 10, { parent = "v4_inv_enabled" })
menu.add_checkbox("Aimbot", "Inventory Checker", "v4_inv_valuables", "Show Valuables", true, { parent = "v4_inv_enabled" })
menu.add_slider_int("Aimbot", "Inventory Checker", "v4_inv_max_valuables", "Max Valuables", 1, 20, 10, { parent = "v4_inv_enabled" })
menu.add_checkbox("Aimbot", "Inventory Checker", "v4_inv_other", "Show Other", false, { parent = "v4_inv_enabled" })
menu.add_slider_int("Aimbot", "Inventory Checker", "v4_inv_max_other", "Max Other", 1, 50, 5, { parent = "v4_inv_enabled" })
menu.add_checkbox("Aimbot", "Inventory Checker", "v4_inv_equipped", "Show Equipped Weapon", true, { parent = "v4_inv_enabled" })
menu.add_checkbox("Aimbot", "Inventory Checker", "v4_inv_icons", "Show Icons", true, { parent = "v4_inv_enabled" })
menu.add_slider_int("Aimbot", "Inventory Checker", "v4_inv_icon_size", "Icon Size", 16, 64, 24, { parent = "v4_inv_enabled" })

-- ============ AIMBOT — TARGET HUD ============
menu.add_group("Aimbot", "Target HUD")
menu.add_checkbox("Aimbot", "Target HUD", "v4_hud_enabled", "Enable Target HUD", false)
menu.add_checkbox("Aimbot", "Target HUD", "v4_hud_name", "Show Name", true, { parent = "v4_hud_enabled" })
menu.add_checkbox("Aimbot", "Target HUD", "v4_hud_weapon", "Show Weapon", true, { parent = "v4_hud_enabled" })
menu.add_checkbox("Aimbot", "Target HUD", "v4_hud_hp", "Show HP", true, { parent = "v4_hud_enabled" })
menu.add_checkbox("Aimbot", "Target HUD", "v4_hud_helmet", "Show Helmet", true, { parent = "v4_hud_enabled" })
menu.add_checkbox("Aimbot", "Target HUD", "v4_hud_armor", "Show Armor (Rig/Vest)", true, { parent = "v4_hud_enabled" })
menu.add_checkbox("Aimbot", "Target HUD", "v4_hud_mask", "Show Mask", true, { parent = "v4_hud_enabled" })
menu.add_checkbox("Aimbot", "Target HUD", "v4_hud_gloves", "Show Gloves", true, { parent = "v4_hud_enabled" })
menu.add_checkbox("Aimbot", "Target HUD", "v4_hud_backpack", "Show Backpack", true, { parent = "v4_hud_enabled" })
menu.add_checkbox("Aimbot", "Target HUD", "v4_hud_dist", "Show Distance", true, { parent = "v4_hud_enabled" })
menu.add_checkbox("Aimbot", "Target HUD", "v4_hud_visible", "Show Visible/Hidden", true, { parent = "v4_hud_enabled" })
menu.add_checkbox("Aimbot", "Target HUD", "v4_hud_icons", "Show Icons", true, { parent = "v4_hud_enabled" })
menu.add_slider_int("Aimbot", "Target HUD", "v4_hud_icon_size", "Icon Size", 16, 64, 24, { parent = "v4_hud_enabled" })
menu.add_slider_int("Aimbot", "Target HUD", "v4_hud_range", "Range (m)", 10, 1500, 300, { parent = "v4_hud_enabled" })
menu.add_slider_int("Aimbot", "Target HUD", "v4_hud_offset_y", "Offset Y", 0, 500, 120, { parent = "v4_hud_enabled" })

-- ============ VISUALS — RADAR ============
menu.add_group("Visuals", "Radar")
menu.add_checkbox("Visuals", "Radar", "v4_radar_enabled", "Enable Radar", false)
menu.add_slider_int("Visuals", "Radar", "v4_radar_size", "Size", 100, 300, 180, { parent = "v4_radar_enabled" })
menu.add_slider_int("Visuals", "Radar", "v4_radar_range", "Range (m)", 50, 500, 150, { parent = "v4_radar_enabled" })
menu.add_checkbox("Visuals", "Radar", "v4_radar_rotate", "Rotate with camera", true, { parent = "v4_radar_enabled" })

-- ============ MISC ============
menu.add_group("Misc", "Misc")
menu.add_checkbox("Misc", "Misc", "v4_hitmarker", "Hitmarker", true)
menu.add_checkbox("Misc", "Misc", "v4_nograss", "No Grass", false)

-- ============ CONFIG ============
menu.add_group("Config", "UI — Colors")
menu.add_hotkey("Config", "UI — Colors", "v4_ui_toggle_key", "Menu Toggle Key", 0x2D)
menu.add_colorpicker("Config", "UI — Colors", "v4_ui_accent", "Accent Color", {0.25, 0.60, 1.00, 1.00})
menu.add_slider_int("Config", "UI — Colors", "v4_ui_opacity", "UI Opacity (%)", 30, 100, 96)
menu.add_slider_int("Config", "UI — Colors", "v4_ui_border_glow", "Border Glow (%)", 0, 200, 100)
menu.add_slider_int("Config", "UI — Colors", "v4_ui_w", "Window Width", 500, 1400, 820)
menu.add_slider_int("Config", "UI — Colors", "v4_ui_h", "Window Height", 400, 1200, 950)
menu.add_button("Config", "UI — Colors", "v4_ui_reset", "Reset Theme", function()
    base_ui.state.ui_settings.accent = {0.25, 0.60, 1.00, 1.00}
    base_ui.state.ui_settings.opacity = 96
    base_ui.state.ui_settings.border_glow = 100
    base_ui.state.ui_settings.window_w = 820
    base_ui.state.ui_settings.window_h = 950
    base_ui:apply_ui_settings()
    base_ui.state.colors["v4_ui_accent"] = {0.25, 0.60, 1.00, 1.00}
    base_ui.state.values["v4_ui_opacity"] = 96
    base_ui.state.values["v4_ui_border_glow"] = 100
    base_ui.state.values["v4_ui_w"] = 820
    base_ui.state.values["v4_ui_h"] = 950
end)

menu.set_callback("v4_ui_toggle_key", function(k)
    base_ui.state.toggle_key = k
    _prev_toggle = false
end)

menu.set_callback("v4_ui_accent", function(c)
    base_ui.state.ui_settings.accent = c
    base_ui:apply_ui_settings()
end)
menu.set_callback("v4_ui_opacity", function(v)
    base_ui.state.ui_settings.opacity = v
    base_ui:apply_ui_settings()
end)
menu.set_callback("v4_ui_border_glow", function(v)
    base_ui.state.ui_settings.border_glow = v
    base_ui:apply_ui_settings()
end)
menu.set_callback("v4_ui_w", function(v)
    base_ui.state.ui_settings.window_w = v
    base_ui:apply_ui_settings()
end)
menu.set_callback("v4_ui_h", function(v)
    base_ui.state.ui_settings.window_h = v
    base_ui:apply_ui_settings()
end)

menu.add_group("Config", "Config")

local CONFIG_NAME = "delta_v2_config.txt"
local function get_config_path()
    local ad = os.getenv("LOCALAPPDATA")
    if ad then return ad .. "\\Project Vector\\Scripts\\" .. CONFIG_NAME end
    return CONFIG_NAME
end

local function m_get(id)
    local ok, v = pcall(function() return menu.get(id) end)
    return ok and v or false
end
local function m_col(id)
    local ok, v = pcall(function() return menu.get_color(id) end)
    return ok and v or {1,1,1,1}
end
local function m_key(id)
    local ok, v = pcall(function() return menu.get_key(id) end)
    return ok and v or 0
end

local STUDS_PER_M = 1 / 0.28
local M_PER_STUDS = 0.28
local ICON_BASE_URL = "https://raw.githubusercontent.com/ericpavva/delta-itemss/main/"

local ITEM_ICONS = {
    ["6B2"] = "6b2.png", ["6B23"] = "6b23.png", ["6B27"] = "6b27.png",
    ["6B43"] = "6b45.png", ["6B45"] = "6b45.png", ["6B47"] = "6b47.png",
    ["6B5"] = "6b5.png", ["Altyn"] = "altyn.png", ["Altyn Helmet"] = "altyn.png",
    ["Balaclava"] = "balaclava.png", ["Bandolier"] = "bandolier.png",
    ["CombatGloves"] = "combatgloves.png", ["Combat Gloves"] = "combatgloves.png",
    ["Concealed Vest"] = "concealedvest.png", ["ConcealedVest"] = "concealedvest.png",
    ["Crown"] = "crown.png", ["Dozer"] = "dozer.png", ["DozerArmor"] = "dozer.png",
    ["Fast MT"] = "fastmt.png", ["FastMT"] = "fastmt.png", ["Fast Mt"] = "fastmt.png",
    ["GP-5"] = "gp5.png", ["GP5"] = "gp5.png", ["GP-7"] = "gp7.png", ["GP7"] = "gp7.png",
    ["HSPV"] = "hspv.png", ["HandWraps"] = "handwraps.png", ["Hand Wraps"] = "handwraps.png",
    ["Head Mount"] = "headmount.png", ["HeadMount"] = "headmount.png",
    ["Improved Outer Lower"] = "improvedouterlower.png",
    ["Improved Outer Lower Armor"] = "improvedouterlower.png",
    ["ImprovedOuterLower"] = "improvedouterlower.png",
    ["Improved Outer"] = "improvedouter.png",
    ["Improved Outer Tactical Vest"] = "improvedouter.png",
    ["ImprovedOuter"] = "improvedouter.png",
    ["JPC"] = "jpc.png", ["KneePads"] = "kneepads.png", ["Knee Pads"] = "kneepads.png",
    ["Kora Kulon"] = "korakulon.png", ["KoraKulon"] = "korakulon.png",
    ["Kulon"] = "korakulon.png", ["Korund"] = "korakulon.png",
    ["Motorcycle"] = "motorcycle.png", ["Motorcycle Helmet"] = "motorcycle.png",
    ["MotorcycleHelmet"] = "motorcycle.png", ["One-Strap"] = "onestrap.png",
    ["One-Strap Backpack Lynx 10L"] = "onestrap.png", ["OneStrap"] = "onestrap.png",
    ["Lynx"] = "onestrap.png", ["Attak5"] = "attak5.png", ["Attak"] = "attak5.png",
    ["Raid Backpack"] = "attak5.png", ["Raid Backpack Attak-5 60L"] = "attak5.png",
    ["SSH-68"] = "ssh68.png", ["SSH68"] = "ssh68.png", ["SSH 68"] = "ssh68.png",
    ["Scav King"] = "scavking.png", ["ScavKingChestplate"] = "scavking.png",
    ["ScavKing"] = "scavking.png", ["Smersh"] = "smersh.png",
    ["SpecopsBackpack"] = "specops.png",
    ["Special Operation Backpack"] = "specops.png",
    ["SpecialOperationBackpack"] = "specops.png",
    ["Specops"] = "specops.png", ["TOR-S"] = "tors.png", ["TORS"] = "tors.png",
    ["Tanker"] = "tanker.png", ["Tanker Helmet"] = "tanker.png",
    ["Tortilla"] = "tortilla.png", ["UNO helmet"] = "unohelmet.png",
    ["UNOhelmet"] = "unohelmet.png", ["UNOVest"] = "unovest.png",
    ["Uno Vest"] = "unovest.png", ["Wasteland Backpack"] = "wastelandbackpack.png",
    ["WastelandBackpack"] = "wastelandbackpack.png",
    ["ZSh-1-2M"] = "zsh.png", ["ZSh"] = "zsh.png", ["ZSh1"] = "zsh.png",
    ["ZSh12M"] = "zsh.png", ["Pantsir"] = "pantsir3.png", ["Pantsir3"] = "pantsir3.png",
    ["Pantsir-3"] = "pantsir3.png",
    ["RPG7"] = "rpg7.png", ["TFZ0"] = "tfz0.png", ["TFZ98S"] = "tfz98.png",
    ["TFZ98"] = "tfz98.png", ["R700"] = "r700.png", ["Saiga"] = "saiga.png",
    ["IZH81"] = "izh81.png", ["IZH12"] = "izh12.png", ["PKM"] = "pkm.png",
    ["SVD"] = "svd.png", ["Mosin"] = "mosin.png", ["FAL"] = "fal.png",
    ["AKMN"] = "akmn.png", ["SKS"] = "sks.png", ["AKM"] = "akm.png",
    ["M4"] = "m4.png", ["M4A1"] = "m4a1.png", ["ADAR15"] = "adar15.png",
    ["AsVal"] = "asval.png", ["Groza"] = "groza.png", ["MP5SD"] = "mp5sd.png",
    ["PPSH41"] = "ppsh41.png", ["TOZ106"] = "toz106.png", ["MK23"] = "mk23.png",
    ["MP443"] = "mp443.png", ["VZ61"] = "vz61.png", ["Makarov"] = "makarov.png",
    ["TT33"] = "tt33.png", ["DV2"] = "dv2.png",
    ["AnarchyTomahawk"] = "anarchytomahawk.png",
    ["Karambit"] = "karambit.png", ["Greatsword"] = "greatsword.png",
}

local ICON_CACHE = {}
local ICON_MISS = {}

local function get_icon_handle(item_name)
    if not item_name then return nil end
    if ICON_MISS[item_name] then return nil end
    local file = ITEM_ICONS[item_name]
    if not file then
        local n = item_name:lower()
        for key, f in pairs(ITEM_ICONS) do
            if key:lower():find(n, 1, true) then file = f break end
        end
    end
    if not file then ICON_MISS[item_name] = true return nil end
    if ICON_CACHE[file] ~= nil then return ICON_CACHE[file] end
    local ok, handle = pcall(function() return draw.LoadImage(ICON_BASE_URL .. file) end)
    if ok and handle then ICON_CACHE[file] = handle return handle end
    ICON_CACHE[file] = false
    return nil
end

local ITEM_NAME_MAP = {
    ["6B43"] = "6B45", ["6b43"] = "6B45", ["6b45"] = "6B45",
    ["TFZ98"] = "TFZ98S", ["tfz98"] = "TFZ98S", ["tfz98s"] = "TFZ98S", ["TFZ98s"] = "TFZ98S",
}

local SLOT_NAMES = {
    "clothingheadware", "clothingchestrig", "clothingmask",
    "clothinggloves", "clothingbackpack", "clothinglegarmor",
    "clothingshirt", "clothingpants", "clothingsack"
}

local BULLET_SPEED = {
    ["12ga"]=425, ["9x39"]=424, ["762x39"]=715, ["556x45"]=933,
    ["762x54"]=885, ["762x51"]=820, ["338lm"]=992, ["9x18"]=359,
    ["9x19"]=465, ["45super"]=465, ["762x25"]=460, ["127x108"]=820,
}

local WEAPON_CALIBER = {
    ["AKMN"]="762x39", ["AKM"]="762x39", ["SKS"]="762x39",
    ["M4"]="556x45", ["M4A1"]="556x45", ["ADAR15"]="556x45",
    ["PKM"]="762x54", ["SVD"]="762x54", ["Mosin"]="762x54",
    ["FAL"]="762x51", ["R700"]="338lm", ["TFZ98S"]="338lm", ["TFZ0"]="9x18",
    ["AsVal"]="9x39", ["VAL"]="9x39", ["MP443"]="9x18", ["Makarov"]="9x18",
    ["MP5"]="9x19", ["MP5SD"]="9x19", ["PPSH41"]="762x25",
    ["Saiga"]="12ga", ["TOZ106"]="12ga", ["IZH12"]="12ga", ["IZH81"]="12ga",
    ["MK23"]="45super", ["RPG7"]="127x108", ["Groza"]="762x39", ["VZ61"]="9x19", ["TT33"]="9x18",
}

local GRAVITY_STUDS = 80.0
local GRAVITY_MS2 = GRAVITY_STUDS * M_PER_STUDS

local function get_bullet_speed(weapon_name)
    if not weapon_name or weapon_name == "" then return 715 end
    local w = weapon_name:gsub("%s+", "")
    for k, v in pairs(WEAPON_CALIBER) do
        if w:lower():find(k:lower(), 1, true) then return BULLET_SPEED[v] or 715 end
    end
    return 715
end

local function calc_ballistic_drop(dist_m, speed_ms, gravity_ms2)
    if not speed_ms or speed_ms <= 0 then speed_ms = 715 end
    if not gravity_ms2 or gravity_ms2 <= 0 then gravity_ms2 = GRAVITY_MS2 end
    local t = dist_m / speed_ms
    return 0.5 * gravity_ms2 * t * t, t
end

local function resolve_item_name(objvalue)
    if not objvalue then return nil end
    local ok, val = pcall(function() return objvalue.value end)
    if ok and val then
        local ok2, n = pcall(function() return val.name end)
        if ok2 and n and n ~= "" then
            n = n:gsub("%s*%(%d+x?%)%s*$", "")
            n = n:gsub("%s*%(%d+%)%s*$", "")
            local nl = n:lower()
            for _, s in ipairs(SLOT_NAMES) do
                if nl == s then return nil end
            end
            n = ITEM_NAME_MAP[n] or n
            return n
        end
    end
    local ok3, n2 = pcall(function() return objvalue.name end)
    if ok3 and n2 and n2 ~= "" then
        n2 = n2:gsub("%s*%(%d+x?%)%s*$", "")
        n2 = n2:gsub("%s*%(%d+%)%s*$", "")
        local nl2 = n2:lower()
        for _, s in ipairs(SLOT_NAMES) do
            if nl2 == s then return nil end
        end
        n2 = ITEM_NAME_MAP[n2] or n2
        return n2
    end
    return nil
end

local function get_equipped_weapon(player_obj)
    if not player_obj then return nil end
    local ok, char = pcall(function() return player_obj.character end)
    if not ok or not char then return nil end
    local ok2, holding = pcall(function() return char:find_first_child("Holding") end)
    if not ok2 or not holding then return nil end
    local ok3, val = pcall(function() return holding.value end)
    if not ok3 or not val then return nil end
    local ok4, name = pcall(function() return val.name end)
    if not ok4 or not name then return nil end
    return ITEM_NAME_MAP[name] or name
end

local SKELETON_PAIRS = {
    {"Head","UpperTorso"},{"UpperTorso","LowerTorso"},
    {"UpperTorso","LeftUpperArm"},{"UpperTorso","RightUpperArm"},
    {"LeftUpperArm","LeftLowerArm"},{"RightUpperArm","RightLowerArm"},
    {"LeftLowerArm","LeftHand"},{"RightLowerArm","RightHand"},
    {"LowerTorso","LeftUpperLeg"},{"LowerTorso","RightUpperLeg"},
    {"LeftUpperLeg","LeftLowerLeg"},{"RightUpperLeg","RightLowerLeg"},
    {"LeftLowerLeg","LeftFoot"},{"RightLowerLeg","RightFoot"}
}

local world = { cam_x=0, cam_y=0, cam_z=0 }
local folders = { ws=nil, drops=nil, ai_zones=nil, vehicles=nil, containers=nil, quests=nil, last=0 }
local npcs_cached = {}
local corpses_cached = {}
local exit_points = {}
local car_points = {}
local container_points = {}
local quest_points = {}
local claymore_points = {}
local boss_cached = { estonia = {}, city13 = {} }
local floor_calc = math.floor
local cached_players_frame = {}
local original_grass_length = nil

local CAR_NAMES = {
    "uaz", "vaz", "vaz2108", "vaz-2108", "niva", "lada", "kamaz", "gaz", "zil",
    "truck", "jeep", "vehicle"
}

local function is_car_name(name)
    if not name then return false end
    local n = name:lower()
    for _, k in ipairs(CAR_NAMES) do
        if n:find(k, 1, true) then return true end
    end
    return false
end

local CAR_CHECK_CACHE = {}

local function is_interactive_car(model, children)
    if not model then return false end
    local addr = nil
    pcall(function() addr = model.address end)
    if addr and CAR_CHECK_CACHE[addr] ~= nil then return CAR_CHECK_CACHE[addr] end
    if model.class_name ~= "Model" then
        if addr then CAR_CHECK_CACHE[addr] = false end
        return false
    end
    if not children then
        local ok_ch, ch = pcall(function() return model:get_children() end)
        if not ok_ch or not ch then
            if addr then CAR_CHECK_CACHE[addr] = false end
            return false
        end
        children = ch
    end
    local n_children = #children
    if n_children < 3 then
        if addr then CAR_CHECK_CACHE[addr] = false end
        return false
    end
    local pp = nil
    pcall(function() pp = model.primary_part end)
    if not pp then
        if addr then CAR_CHECK_CACHE[addr] = false end
        return false
    end
    local has_seat = false
    local has_wheel = false
    local has_engine = false
    local wheel_count = 0
    for i = 1, n_children do
        local c = children[i]
        if c then
            local ccn = c.class_name
            if ccn == "VehicleSeat" or ccn == "Seat" then
                has_seat = true
            else
                local cn = c.name
                if cn then
                    local cl = cn:lower()
                    if cl:find("wheel", 1, true) then
                        has_wheel = true
                        wheel_count = wheel_count + 1
                    end
                    if not has_engine and cl:find("engine", 1, true) then has_engine = true end
                end
            end
        end
    end
    local result = has_seat or (has_wheel and wheel_count >= 3) or (has_wheel and has_engine)
    if addr then CAR_CHECK_CACHE[addr] = result end
    return result
end

local function is_valid(inst)
    if not inst then return false end
    local ok, p = pcall(function() return inst.parent end)
    return ok and p ~= nil
end

local function dist3(ax,ay,az,bx,by,bz)
    local dx,dy,dz = ax-bx, ay-by, az-bz
    return sqrt(dx*dx+dy*dy+dz*dz)
end

local function refresh_folders()
    local now = utility.get_tick_count()
    if folders.ws and (now - folders.last) < 2000 then return true end
    folders.last = now
    local ws = game.workspace
    if not ws then return false end
    if folders.ws ~= ws or not is_valid(folders.ws) then
        folders.ws = ws
        folders.drops=nil folders.ai_zones=nil folders.vehicles=nil
        folders.containers=nil folders.quests=nil
    end
    if not folders.drops or not is_valid(folders.drops) then
        folders.drops = ws:find_first_child("DroppedItems")
    end
    if not folders.ai_zones or not is_valid(folders.ai_zones) then
        folders.ai_zones = ws:find_first_child("AiZones")
    end
    if not folders.vehicles or not is_valid(folders.vehicles) then
        folders.vehicles = ws:find_first_child("Vehicles")
    end
    if not folders.containers or not is_valid(folders.containers) then
        folders.containers = ws:find_first_child("Containers")
    end
    if not folders.quests or not is_valid(folders.quests) then
        folders.quests = ws:find_first_child("QuestItems")
    end
    return true
end

local BOSS_TARGETS_ESTONIA = {
    {key = "anton", label = "Anton", names = {"anton"}, max_hp = 400, kind = "boss"},
    {key = "dozer", label = "Dozer", names = {"dozer"}, max_hp = 400, kind = "boss"},
    {key = "whisper", label = "Whisper", names = {"whisper"}, max_hp = 400, kind = "boss"},
    {key = "mi24v", label = "MI24V", names = {"mi24v"}, max_hp = 0, kind = "vehicle"},
}
local BOSS_TARGETS_CITY13 = {
    {key = "scavking", label = "ScavKing", names = {"scavking"}, max_hp = 0, kind = "boss"},
    {key = "btr80", label = "BTR80", names = {"btr80"}, max_hp = 0, kind = "vehicle"},
}

local function name_matches_boss(name, boss)
    if not name then return false end
    local n = name:lower()
    for _, alias in ipairs(boss.names) do
        if n == alias then return true end
    end
    return false
end

local function scan_one_target(boss)
    local entry = { label = boss.label, found = false, alive = false, hp = 0, max_hp = boss.max_hp, hrp = nil, kind = boss.kind }
    local ai = folders.ai_zones
    if not ai then return entry end
    local zones = ai:get_children()
    if not zones then return entry end
    for _, zone in ipairs(zones) do
        if zone.class_name == "Folder" then
            local npcs = zone:get_children()
            if npcs then
                for _, npc in ipairs(npcs) do
                    if npc.class_name == "Model" and name_matches_boss(npc.name, boss) then
                        entry.found = true
                        local hum = npc:find_first_child_of_class("Humanoid")
                        local hrp = npc:find_first_child("HumanoidRootPart")
                        if not hrp then hrp = npc:find_first_child_of_class("BasePart") end
                        entry.hrp = hrp
                        local hp, maxhp = 0, boss.max_hp
                        if hum then
                            pcall(function() hp = hum.health end)
                            pcall(function() maxhp = hum.max_health end)
                        end
                        entry.hp = hp or 0
                        entry.max_hp = maxhp or boss.max_hp
                        if hum then entry.alive = (hp or 0) > 0 else entry.alive = true end
                        return entry
                    end
                end
            end
        end
    end
    return entry
end

local function scan_bosses()
    local est, city = {}, {}
    for _, boss in ipairs(BOSS_TARGETS_ESTONIA) do est[boss.key] = scan_one_target(boss) end
    for _, boss in ipairs(BOSS_TARGETS_CITY13) do city[boss.key] = scan_one_target(boss) end
    boss_cached = { estonia = est, city13 = city }
end

local function scan_npcs()
    npcs_cached = {}
    if not folders.ai_zones then return end
    local ok, zones = pcall(function() return folders.ai_zones:get_children() end)
    if not ok or not zones then return end
    for i=1,#zones do
        local zone = zones[i]
        if zone.class_name == "Folder" then
            local ok2, npcs = pcall(function() return zone:get_children() end)
            if ok2 and npcs then
                for j=1,#npcs do
                    local npc = npcs[j]
                    if npc.class_name == "Model" and npc.name ~= "" then
                        local hum = npc:find_first_child_of_class("Humanoid")
                        local head = npc:find_first_child("Head")
                        if hum and head then
                            local hp = hum.health
                            if hp and hp > 0 then
                                local parts = {}
                                local ok3, ch = pcall(function() return npc:get_children() end)
                                if ok3 and ch then
                                    for k=1,#ch do
                                        local p = ch[k]
                                        if p:is_a("BasePart") then
                                            local sz = p.size
                                            if sz then
                                                parts[p.name] = {part=p, hx=sz.x*0.5, hy=sz.y*0.5, hz=sz.z*0.5}
                                            end
                                        end
                                    end
                                end
                                npcs_cached[#npcs_cached+1] = {name=npc.name, hum=hum, head=head, parts=parts, model=npc}
                            end
                        end
                    end
                end
            end
        end
    end
end

local function scan_corpses()
    corpses_cached = {}
    local drops = folders.drops
    if not drops then return end
    local ok, kids = pcall(function() return drops:get_children() end)
    if not ok or not kids then return end
    local function process(model)
        if not model or model.class_name ~= "Model" then return end
        local head = model:find_first_child("Head")
        local torso = model:find_first_child("UpperTorso")
        if not head and not torso then return end
        local ref = head or torso
        local pos = ref.position
        if not pos then return end
        corpses_cached[#corpses_cached+1] = {name = model.name, part = ref, model = model}
    end
    for i=1,#kids do
        local child = kids[i]
        if child.class_name == "Model" then
            process(child)
        elseif child.class_name == "Folder" then
            local ok2, subs = pcall(function() return child:get_children() end)
            if ok2 and subs then for j=1,#subs do process(subs[j]) end end
        end
    end
end

local function scan_exits()
    exit_points = {}
    local ws = game.workspace
    if not ws then return end
    local nc = ws:find_first_child("NoCollision")
    if not nc then return end
    local ex = nc:find_first_child("ExitLocations")
    if not ex then return end
    local ok, kids = pcall(function() return ex:get_children() end)
    if not ok or not kids then return end
    for i=1,#kids do
        local c = kids[i]
        local pos = c.position
        if pos then exit_points[#exit_points+1] = {name = "Exit " .. (#exit_points + 1), part = c} end
    end
end

local function scan_cars()
    car_points = {}
    local ws = game.workspace
    if not ws then return end
    local seen = {}
    local function try_car(model)
        if not model or model.class_name ~= "Model" then return end
        local addr = nil
        pcall(function() addr = model.address end)
        if addr and seen[addr] then return end
        if addr then seen[addr] = true end
        local ok_ch, children = pcall(function() return model:get_children() end)
        if not ok_ch or not children then return end
        if not is_interactive_car(model, children) then return end
        local part = nil
        for i = 1, #children do
            local c = children[i]
            if c and c:is_a("BasePart") then part = c break end
        end
        if not part then part = model:find_first_descendant_of_class("BasePart") end
        if not part or not part.position then return end
        car_points[#car_points+1] = {name = "Car", part = part, model = model}
    end
    local veh = folders.vehicles
    if veh then
        local ok, kids = pcall(function() return veh:get_children() end)
        if ok and kids then for i = 1, #kids do try_car(kids[i]) end end
    end
    local ok2, ws_kids = pcall(function() return ws:get_children() end)
    if ok2 and ws_kids then
        for i = 1, #ws_kids do
            local c = ws_kids[i]
            if c and c.class_name == "Model" and is_car_name(c.name) then try_car(c) end
        end
    end
end

local function get_container_contents(model)
    local inv = model:find_first_child("Inventory")
    if not inv then return nil end
    local ok, children = pcall(function() return inv:get_children() end)
    if not ok or not children or #children == 0 then return nil end
    local counts = {}
    for i=1,#children do
        local it = children[i]
        local name = it.name
        if it.class_name == "ObjectValue" and it.value then
            local ok2, vn = pcall(function() return it.value.name end)
            if ok2 then name = vn end
        end
        if name ~= "" then counts[name] = (counts[name] or 0) + 1 end
    end
    local out = {}
    for name, count in pairs(counts) do
        out[#out+1] = count > 1 and (name .. " x" .. count) or name
    end
    return #out > 0 and out or nil
end

local function scan_containers()
    container_points = {}
    local cont = folders.containers
    if not cont then return end
    local ok, kids = pcall(function() return cont:get_children() end)
    if not ok or not kids then return end
    local function process(model)
        if not model or model.class_name ~= "Model" or model.name == "" then return end
        local part = model:find_first_child_of_class("BasePart")
        if not part then return end
        local contents = get_container_contents(model)
        container_points[#container_points+1] = {name = model.name, part = part, contents = contents}
    end
    for i=1,#kids do
        local c = kids[i]
        if c.class_name == "Model" then
            process(c)
        elseif c.class_name == "Folder" then
            local ok2, subs = pcall(function() return c:get_children() end)
            if ok2 and subs then for j=1,#subs do process(subs[j]) end end
        end
    end
end

local function scan_quests()
    quest_points = {}
    local q = folders.quests
    if not q then return end
    local ok, kids = pcall(function() return q:get_children() end)
    if not ok or not kids then return end
    local function process(model)
        if not model or model.class_name ~= "Model" or model.name == "" then return end
        local part = model:find_first_child_of_class("BasePart")
        if not part then return end
        quest_points[#quest_points+1] = {name = model.name, part = part}
    end
    for i=1,#kids do
        local c = kids[i]
        if c.class_name == "Model" then
            process(c)
        elseif c.class_name == "Folder" then
            local ok2, subs = pcall(function() return c:get_children() end)
            if ok2 and subs then for j=1,#subs do process(subs[j]) end end
        end
    end
end

local function scan_claymores()
    claymore_points = {}
    local ai = folders.ai_zones
    if not ai then return end
    local ok, zones = pcall(function() return ai:get_children() end)
    if not ok or not zones then return end
    for _, zone in ipairs(zones) do
        local zn = (zone.name or ""):lower()
        if zn:find("claymore", 1, true) or zn:find("landmine", 1, true) then
            local ok2, kids = pcall(function() return zone:get_children() end)
            if ok2 and kids then
                for _, model in ipairs(kids) do
                    if model.class_name == "Model" then
                        local part = model:find_first_child_of_class("BasePart")
                        if part and part.position then
                            claymore_points[#claymore_points+1] = {name = model.name, part = part, model = model}
                        end
                    end
                end
            end
        end
    end
end

local function get_bounds_from_parts(parts)
    local minx,miny,maxx,maxy = 10000,10000,-10000,-10000
    local valid = false
    for _, d in pairs(parts) do
        if d.part then
            local p = d.part.position
            if p then
                for ox=-1,1,2 do for oy=-1,1,2 do for oz=-1,1,2 do
                    local sx,sy,vis = draw.world_to_screen(p.x+d.hx*ox, p.y+d.hy*oy, p.z+d.hz*oz)
                    if vis then
                        valid = true
                        if sx<minx then minx=sx end if sx>maxx then maxx=sx end
                        if sy<miny then miny=sy end if sy>maxy then maxy=sy end
                    end
                end end end
            end
        end
    end
    if valid then return {x=minx,y=miny,w=maxx-minx,h=maxy-miny} end
    return nil
end

local function get_bounds_from_player(p)
    local b = p:GetBounds()
    if b and b.valid then return b end
    return nil
end

local function update_camera()
    local pos = camera.get_position()
    if pos then world.cam_x=pos.x world.cam_y=pos.y world.cam_z=pos.z end
end

local S = {}

local function refresh_settings()
    S.player          = m_get("v4_player_enabled")
    S.player_box      = m_get("v4_player_box")
    S.player_box_col  = m_col("v4_player_box")
    S.player_hp       = m_get("v4_player_health")
    S.player_name     = m_get("v4_player_name")
    S.player_name_col = m_col("v4_player_name")
    S.player_dist     = m_get("v4_player_dist")
    S.player_dist_col = m_col("v4_player_dist")
    S.player_skel     = m_get("v4_player_skeleton")
    S.player_skel_col = m_col("v4_player_skeleton")
    S.player_weapon   = m_get("v4_player_weapon")
    S.player_weapon_col = m_col("v4_player_weapon")
    S.player_team     = m_get("v4_player_team_check")
    S.player_range    = m_get("v4_player_range") or 500
    S.player_range_studs = S.player_range * STUDS_PER_M
    S.npc             = m_get("v4_npc_enabled")
    S.npc_box         = m_get("v4_npc_box")
    S.npc_box_col     = m_col("v4_npc_box")
    S.npc_hp          = m_get("v4_npc_health")
    S.npc_name        = m_get("v4_npc_name")
    S.npc_name_col    = m_col("v4_npc_name")
    S.npc_dist        = m_get("v4_npc_dist")
    S.npc_dist_col    = m_col("v4_npc_dist")
    S.npc_skel        = m_get("v4_npc_skeleton")
    S.npc_skel_col    = m_col("v4_npc_skeleton")
    S.npc_range       = m_get("v4_npc_range") or 140
    S.npc_range_studs = S.npc_range * STUDS_PER_M
    S.car             = m_get("v4_car_enabled")
    S.car_col         = m_col("v4_car_col")
    S.car_range       = m_get("v4_car_range") or 500
    S.car_range_studs = S.car_range * STUDS_PER_M
    S.exit_enabled    = m_get("v4_exit_enabled")
    S.exit_col        = m_col("v4_exit_col")
    S.exit_range      = m_get("v4_exit_range") or 300
    S.exit_range_studs = S.exit_range * STUDS_PER_M
    S.loot            = m_get("v4_loot_enabled")
    S.loot_weapons    = m_get("v4_loot_weapons")
    S.loot_armor      = m_get("v4_loot_armor")
    S.loot_valuables  = m_get("v4_loot_valuables")
    S.loot_meds       = m_get("v4_loot_meds")
    S.loot_ammo       = m_get("v4_loot_ammo")
    S.loot_other      = m_get("v4_loot_other")
    S.loot_prefix     = m_get("v4_loot_prefix")
    S.loot_range      = m_get("v4_loot_range") or 84
    S.loot_range_studs = S.loot_range * STUDS_PER_M
    S.corpse          = m_get("v4_corpse_enabled")
    S.corpse_name     = m_get("v4_corpse_name")
    S.corpse_name_col = m_col("v4_corpse_name")
    S.corpse_dist     = m_get("v4_corpse_dist")
    S.corpse_dist_col = m_col("v4_corpse_dist")
    S.corpse_marker   = m_get("v4_corpse_marker")
    S.corpse_marker_col = m_col("v4_corpse_marker")
    S.corpse_range    = m_get("v4_corpse_range") or 200
    S.corpse_range_studs = S.corpse_range * STUDS_PER_M
    S.container       = m_get("v4_container_enabled")
    S.container_col   = m_col("v4_container_col")
    S.container_range = m_get("v4_container_range") or 84
    S.container_range_studs = S.container_range * STUDS_PER_M
    S.container_contents = m_get("v4_container_contents")
    S.quest           = m_get("v4_quest_enabled")
    S.quest_col       = m_col("v4_quest_col")
    S.quest_range     = m_get("v4_quest_range") or 140
    S.quest_range_studs = S.quest_range * STUDS_PER_M
    S.claymore        = m_get("v4_claymore_enabled")
    S.claymore_col    = m_col("v4_claymore_col")
    S.claymore_range  = m_get("v4_claymore_range") or 140
    S.claymore_range_studs = S.claymore_range * STUDS_PER_M
    S.claymore_names  = m_get("v4_claymore_names")
    S.chams           = m_get("v4_chams_enabled")
    S.chams_style     = m_get("v4_chams_style") or 0
    S.chams_players   = m_get("v4_chams_players")
    S.chams_npcs      = m_get("v4_chams_npcs")
    S.chams_col       = m_col("v4_chams_color")
    S.chams_gradient  = m_get("v4_chams_gradient")
    S.chams_col2      = m_col("v4_chams_color2")
    S.chams_vis_fov   = m_get("v4_chams_vis_fov") or 90
    S.boss            = m_get("v4_boss_enabled")
    S.lt              = m_get("v4_lt_enabled")
    S.lt_max          = m_get("v4_lt_max") or 50
    S.lt_far          = m_get("v4_lt_far") or 1000
    S.lt_far_studs    = S.lt_far * STUDS_PER_M
    S.aim             = m_get("v4_aim_enabled")
    S.aim_key         = m_key("v4_aim_keybind")
    if S.aim_key == 0 then S.aim_key = 0x02 end
    S.aim_target      = m_get("v4_aim_target") or 0
    local ab = m_get("v4_aim_bone")
    if type(ab) == "number" then S.aim_bone = ab else S.aim_bone = 0 end
    S.aim_fov         = m_get("v4_aim_fov") or 150
    S.aim_smooth      = m_get("v4_aim_smooth") or 4
    S.aim_predict     = m_get("v4_aim_predict")
    S.aim_predict_scale = (m_get("v4_aim_predict_scale") or 120) / 100
    S.aim_lead        = m_get("v4_aim_lead")
    S.aim_visible     = m_get("v4_aim_visible")
    S.aim_draw_fov    = m_get("v4_aim_draw_fov")
    S.aim_lock        = m_get("v4_aim_lock")
    S.inv             = m_get("v4_inv_enabled")
    S.inv_key         = m_key("v4_inv_keybind")
    if S.inv_key == 0 then S.inv_key = 0x02 end
    S.inv_mode        = m_get("v4_inv_mode") or 0
    S.inv_guns        = m_get("v4_inv_guns")
    S.inv_max_guns    = m_get("v4_inv_max_guns") or 10
    S.inv_armor       = m_get("v4_inv_armor")
    S.inv_max_armor   = m_get("v4_inv_max_armor") or 10
    S.inv_valuables   = m_get("v4_inv_valuables")
    S.inv_max_val     = m_get("v4_inv_max_valuables") or 10
    S.inv_other       = m_get("v4_inv_other")
    S.inv_max_other   = m_get("v4_inv_max_other") or 5
    S.inv_equipped    = m_get("v4_inv_equipped")
    S.inv_icons       = m_get("v4_inv_icons")
    S.inv_icon_size   = m_get("v4_inv_icon_size") or 24
    S.hud             = m_get("v4_hud_enabled")
    S.hud_name        = m_get("v4_hud_name")
    S.hud_weapon      = m_get("v4_hud_weapon")
    S.hud_hp          = m_get("v4_hud_hp")
    S.hud_helmet      = m_get("v4_hud_helmet")
    S.hud_armor       = m_get("v4_hud_armor")
    S.hud_mask        = m_get("v4_hud_mask")
    S.hud_gloves      = m_get("v4_hud_gloves")
    S.hud_backpack    = m_get("v4_hud_backpack")
    S.hud_dist        = m_get("v4_hud_dist")
    S.hud_visible     = m_get("v4_hud_visible")
    S.hud_icons       = m_get("v4_hud_icons")
    S.hud_icon_size   = m_get("v4_hud_icon_size") or 24
    S.hud_range       = m_get("v4_hud_range") or 300
    S.hud_range_studs = S.hud_range * STUDS_PER_M
    S.hud_offset_y    = m_get("v4_hud_offset_y") or 120
    S.radar           = m_get("v4_radar_enabled")
    S.radar_size      = m_get("v4_radar_size") or 180
    S.radar_range     = m_get("v4_radar_range") or 150
    S.radar_range_studs = S.radar_range * STUDS_PER_M
    S.radar_rotate    = m_get("v4_radar_rotate")
    S.hitmarker       = m_get("v4_hitmarker")
    S.nograss         = m_get("v4_nograss")
end

local LOOT_CATEGORIES = {
    weapon = {"AsVal","FAL","AKMN","AKM","RPG7","M4","M4A1","ADAR","PKM","R700","SVD","TFZ98S","TFZ98","TFZ0","Saiga","MP443","PPSH41","Mosin","MK23","MP5","TOZ106","IZH12","IZH81","SKS","Makarov","Skorpion","Glock","Groza","VZ61","TT33","Karambit","Scythe","AnarchyTomahawk","Greatsword","DV2"},
    armor = {
        "Motorcycle","Head Mount","SSH-68","Tanker","6B27","UNO helmet","TOR-S","TORS",
        "6B47","ZSh-1-2M","Fast MT","Crown","Altyn","Altyn Helmet","Helmet",
        "Night Vision Goggles 9","NightVisionGoggles",
        "Low Cut Visor","LowCutVisor",
        "Spartan Tech Titan Shield","SpartanTech",
        "Fast MT Visor","FastMTVisor",
        "Quad Night Vision Goggles","QuadNightVision",
        "Altyn Visor","AltynVisor",
        "Maska Visor","MaskaVisor",
        "ZSh-1-2M Visor","ZShVisor","ZSh-1-2MVisor",
        "Motorcycle Helmet Visor","MotorcycleVisor","MotorcycleHelmetVisor",
        "GP-5","GP-7","Gas Mask","Balaclava","SewnMask",
        "Bandolier","Smersh","6B2","Uno Vest","UNOVest","6B23","Concealed Vest",
        "Kora Kulon","Korund","6B5","JPC","Pantsir","Scav King","6B45","6B43","6B46",
        "Improved Outer","Improved Outer Lower","IOTV4","HSPV","Dozer","Juggernaut",
        "TitanShield","Press","Slick","Hexgrid","LBT","M2","Strandhogg","Crye",
        "PACA","Trooper","Lvl4Plate","Armor","Vest","Rig","Chestplate","Carrier",
        "Plate Carrier","Osprey","Defender","Gzhel",
        "One-Strap","Lynx","Wasteland Backpack","Special Operation Backpack",
        "Tortilla","Raid Backpack","Attak","SpecopsBackpack","Backpack","Bag",
        "Knee Pads","KneePads","KneePad",
        "Combat Gloves","CombatGloves","Hand Wraps","HandWraps","Gloves","Glove",
    },
    valuable = {"GPU","Gold","FlareGun","SPSh44","GoldWatch","GoldTicket","Ticket","Mag556Rnd100","Reapir","SOCOM556","RepairKit","Intel","Bitcoin","Key","BlueCard","OrangeCard","AA2","AABattery","AI2","Case","LEDX","GoldSkull","SolterStatue","MPSU","CPU"},
    med = {"Medkit","Bandage","Splint","Painkiller","Pill","Salewa","Augmentin","IFAK","Car","Surv12","MedBag","Surgical"},
    ammo = {"Ammo","Round","Mag","Bullet","762x","556x","9x19","9x39","12ga"},
}

local loot_colors = {
    weapon={1.00,0.30,0.30,1}, armor={0.18,0.72,0.84,1},
    valuable={0.35,0.90,0.45,1}, med={1.00,0.30,0.50,1},
    ammo={1.00,0.75,0.20,1}, other={0.55,0.55,0.60,1},
}

local loot_prefix = {
    weapon="[W]", armor="[A]", valuable="[V]", med="[M]", ammo="[B]", other="[.]",
}

local BLACKLIST = {
    "camo pants","camo shirt","camo jacket",
    "wastelandshirt","wastelandpants",
    "summershirt","summerpants",
    "civilianpants","civilianshirt",
    "ghillie",
    "clothingmask","clothingheadwear","clothinggloves",
    "clothingshirt","clothingpants","clothingsack",
    "clothingbackpack","clothinglegarmor","clothingchestrig",
    "hair"
}

local VALUE_BLACKLIST = { "key", "card", "aa2", "ai2", "aabattery" }

local function is_value_blacklisted(name)
    if not name or name == "" then return false end
    local n = name:lower()
    for _, kw in ipairs(VALUE_BLACKLIST) do
        if n:find(kw, 1, true) then return true end
    end
    return false
end

local NOT_WEAPON_PARTS = {
    "flashhider", "muzzle", "suppressor", "silencer", "compensator",
    "stock", "grip", "handguard", "barrel", "bolt", "charginghandle",
    "mag", "scope", "sight", "receiver", "upper", "lower",
    "trigger", "hammer", "safety", "dustcover", "rail", "bipod",
    "laser", "foregrip", "buffer", "spring", "pin",
    "mount", "adapter", "plug", "attachment",
    "front", "rear", "handle", "impr",
    "flashlight", "oilcan",
    "cleaning", "kit", "firemode",
    "ammobox", "100rnd", "20rnd", "30rnd", "40rnd", "60rnd",
    "bullet", "cartridge", "grenade", "shell",
    "visor",
}

local function is_weapon_attachment(name)
    if not name then return false end
    local n = name:lower()
    for _, kw in ipairs(NOT_WEAPON_PARTS) do
        if n:find(kw, 1, true) then return true end
    end
    return false
end

local function categorize(name)
    if not name or name == "" then return "other" end
    local n = name:lower()
    for _, b in ipairs(BLACKLIST) do
        if n:find(b, 1, true) then return "other" end
    end
    if is_value_blacklisted(name) then return "other" end
    for _, cat in ipairs({"armor","weapon","valuable","med","ammo"}) do
        for _, kw in ipairs(LOOT_CATEGORIES[cat]) do
            if n:find(kw:lower(), 1, true) then return cat end
        end
    end
    return "other"
end

local function cat_enabled(cat)
    if cat == "weapon" then return S.loot_weapons end
    if cat == "armor" then return S.loot_armor end
    if cat == "valuable" then return S.loot_valuables end
    if cat == "med" then return S.loot_meds end
    if cat == "ammo" then return S.loot_ammo end
    return S.loot_other
end

local loot_items = {}

local function scan_loot()
    loot_items = {}
    if not folders.drops then return end
    local ok, children = pcall(function() return folders.drops:get_children() end)
    if not ok or not children then return end
    local function process(model)
        if model.class_name ~= "Model" or model.name == "" then return end
        local part = model:find_first_child_of_class("BasePart")
        if not part then return end
        local head = model:find_first_child("Head")
        local torso = model:find_first_child("UpperTorso")
        if head or torso then return end
        loot_items[#loot_items+1] = {name=model.name, part=part}
    end
    for i=1,#children do
        local child = children[i]
        if child.class_name == "Folder" then
            local ok2, subs = pcall(function() return child:get_children() end)
            if ok2 and subs then for j=1,#subs do process(subs[j]) end end
        elseif child.class_name == "Model" then
            process(child)
        end
    end
end

local function draw_player_esp()
    if not S.player then return end
    local players = cached_players_frame
    if #players == 0 then return end
    local lp = entity.get_local_player()
    if not lp then return end
    local local_team = lp.team
    local range_studs = S.player_range_studs
    for i=1,#players do
        local p = players[i]
        if p and p.is_valid and p.is_alive and not p.is_local then
            local team_ok = true
            if S.player_team and p.has_team and local_team then
                if p.team == local_team then team_ok = false end
            end
            if team_ok then
                local pos = p.position
                if pos then
                    local dist = dist3(pos.x,pos.y,pos.z, world.cam_x,world.cam_y,world.cam_z)
                    if dist <= range_studs then
                        local b = get_bounds_from_player(p)
                        if b then
                            local mid = b.x + b.w * 0.5
                            if S.player_box then draw.box(b.x, b.y, b.w, b.h, S.player_box_col, 0) end
                            if S.player_hp then draw.health_bar(b.x - 6, b.y, b.h, p.health, p.max_health) end
                            local y_off = b.y - 16
                            if S.player_name then
                                local tw = draw.get_text_size(p.name, 14)
                                draw.text(mid - tw*0.5, y_off, p.name, S.player_name_col, 14)
                                y_off = y_off - 16
                            end
                            if S.player_weapon then
                                local wname = get_equipped_weapon(p)
                                if wname then
                                    local wtxt = "[" .. wname .. "]"
                                    local tw = draw.get_text_size(wtxt, 12)
                                    draw.text(mid - tw*0.5, y_off, wtxt, S.player_weapon_col, 12)
                                end
                            end
                            if S.player_dist then
                                local dm = floor_calc(dist * M_PER_STUDS)
                                local dtxt = dm .. "m"
                                local tw = draw.get_text_size(dtxt, 12)
                                draw.text(mid - tw*0.5, b.y + b.h + 4, dtxt, S.player_dist_col, 12)
                            end
                            if S.player_skel then
                                local bones = p:GetBonesScreen()
                                if bones then
                                    for j=1,#SKELETON_PAIRS do
                                        local pair = SKELETON_PAIRS[j]
                                        local a = bones[pair[1]]
                                        local sb = bones[pair[2]]
                                        if a and sb then
                                            draw.line(a[1],a[2],sb[1],sb[2], S.player_skel_col, 1.5)
                                        end
                                    end
                                end
                            end
                        end
                    end
                end
            end
        end
    end
end

local function draw_npc_esp()
    if not S.npc or #npcs_cached == 0 then return end
    local range_studs = S.npc_range_studs
    for i=1,#npcs_cached do
        local n = npcs_cached[i]
        if n.head and n.hum then
            local hp = n.hum.health or 0
            local max_hp = n.hum.max_health or 100
            if hp > 0 then
                local hpos = n.head.position
                if hpos then
                    local dist = dist3(hpos.x,hpos.y,hpos.z, world.cam_x,world.cam_y,world.cam_z)
                    if dist <= range_studs then
                        local b = get_bounds_from_parts(n.parts)
                        if b then
                            local mid = b.x + b.w * 0.5
                            if S.npc_box then draw.box(b.x, b.y, b.w, b.h, S.npc_box_col, 0) end
                            if S.npc_hp then draw.health_bar(b.x - 6, b.y, b.h, hp, max_hp) end
                            if S.npc_name then
                                local tw = draw.get_text_size(n.name, 14)
                                draw.text(mid - tw*0.5, b.y - 16, n.name, S.npc_name_col, 14)
                            end
                            if S.npc_dist then
                                local dm = floor_calc(dist * M_PER_STUDS)
                                local dtxt = dm .. "m"
                                local tw = draw.get_text_size(dtxt, 12)
                                draw.text(mid - tw*0.5, b.y + b.h + 4, dtxt, S.npc_dist_col, 12)
                            end
                            if S.npc_skel then
                                local screen = {}
                                for name,pd in pairs(n.parts) do
                                    if pd.part then
                                        local pp = pd.part.position
                                        if pp then
                                            local sx,sy,vis = draw.world_to_screen(pp.x,pp.y,pp.z)
                                            if vis then screen[name] = {sx, sy} end
                                        end
                                    end
                                end
                                for j=1,#SKELETON_PAIRS do
                                    local pair = SKELETON_PAIRS[j]
                                    local a, sb = screen[pair[1]], screen[pair[2]]
                                    if a and sb then draw.line(a[1],a[2],sb[1],sb[2], S.npc_skel_col, 1.5) end
                                end
                            end
                        end
                    end
                end
            end
        end
    end
end

local function draw_car_esp()
    if not S.car or #car_points == 0 then return end
    local range_studs = S.car_range_studs
    for i=1,#car_points do
        local c = car_points[i]
        local part = c.part
        if part then
            local pos = part.position
            if pos then
                local dist = dist3(pos.x,pos.y,pos.z, world.cam_x,world.cam_y,world.cam_z)
                if dist <= range_studs then
                    local sx, sy, vis = draw.world_to_screen(pos.x, pos.y + 3, pos.z)
                    if vis then
                        local dm = floor_calc(dist * M_PER_STUDS)
                        local txt = "Car [" .. dm .. "m]"
                        local tw = draw.get_text_size(txt, 13)
                        draw.text(sx - tw*0.5, sy, txt, S.car_col, 13)
                    end
                end
            end
        end
    end
end

local function draw_exit_esp()
    if not S.exit_enabled or #exit_points == 0 then return end
    local range_studs = S.exit_range_studs
    for i=1,#exit_points do
        local e = exit_points[i]
        local part = e.part
        if part then
            local pos = part.position
            if pos then
                local dist = dist3(pos.x,pos.y,pos.z, world.cam_x,world.cam_y,world.cam_z)
                if dist <= range_studs then
                    local sx, sy, vis = draw.world_to_screen(pos.x, pos.y + 3, pos.z)
                    if vis then
                        local dm = floor_calc(dist * M_PER_STUDS)
                        local txt = e.name .. " [" .. dm .. "m]"
                        local tw = draw.get_text_size(txt, 13)
                        draw.text(sx - tw*0.5, sy, txt, S.exit_col, 13)
                    end
                end
            end
        end
    end
end

local function draw_corpse_esp()
    if not S.corpse or #corpses_cached == 0 then return end
    local range_studs = S.corpse_range_studs
    for i=1,#corpses_cached do
        local d = corpses_cached[i]
        local part = d.part
        if part then
            local pos = part.position
            if pos then
                local dist = dist3(pos.x,pos.y,pos.z, world.cam_x,world.cam_y,world.cam_z)
                if dist <= range_studs then
                    local sx, sy, vis = draw.world_to_screen(pos.x, pos.y + 2, pos.z)
                    if vis then
                        local dm = floor_calc(dist * M_PER_STUDS)
                        local y_off = sy
                        if S.corpse_marker then
                            local sz = 6
                            draw.line(sx - sz, y_off - sz, sx + sz, y_off + sz, S.corpse_marker_col, 2)
                            draw.line(sx + sz, y_off - sz, sx - sz, y_off + sz, S.corpse_marker_col, 2)
                            y_off = y_off + 12
                        end
                        if S.corpse_name then
                            local txt = d.name .. " [DEAD]"
                            local tw = draw.get_text_size(txt, 14)
                            draw.text(sx - tw*0.5, y_off, txt, S.corpse_name_col, 14)
                            y_off = y_off + 16
                        end
                        if S.corpse_dist then
                            local dtxt = dm .. "m"
                            local tw = draw.get_text_size(dtxt, 12)
                            draw.text(sx - tw*0.5, y_off, dtxt, S.corpse_dist_col, 12)
                        end
                    end
                end
            end
        end
    end
end

local function draw_loot_esp()
    if not S.loot or #loot_items == 0 then return end
    local range_studs = S.loot_range_studs
    local any_cat = S.loot_weapons or S.loot_armor or S.loot_valuables
                    or S.loot_meds or S.loot_ammo or S.loot_other
    if not any_cat then return end
    for i=1,#loot_items do
        local entry = loot_items[i]
        local part = entry.part
        if part then
            local pos = part.position
            if pos then
                local dist = dist3(pos.x,pos.y,pos.z, world.cam_x,world.cam_y,world.cam_z)
                if dist <= range_studs then
                    local cat = categorize(entry.name)
                    if cat_enabled(cat) then
                        local sx, sy, vis = draw.world_to_screen(pos.x, pos.y, pos.z)
                        if vis then
                            local dm = floor_calc(dist * M_PER_STUDS)
                            local col = loot_colors[cat] or loot_colors.other
                            local prefix = S.loot_prefix and (loot_prefix[cat] or "") or ""
                            local txt = prefix .. " " .. entry.name .. " [" .. dm .. "m]"
                            local tw = draw.get_text_size(txt, 13)
                            draw.text(sx - tw*0.5, sy, txt, col, 13)
                        end
                    end
                end
            end
        end
    end
end

local function draw_container_esp()
    if not S.container or #container_points == 0 then return end
    local range_studs = S.container_range_studs
    for i=1,#container_points do
        local entry = container_points[i]
        local part = entry.part
        if part then
            local pos = part.position
            if pos then
                local dist = dist3(pos.x,pos.y,pos.z, world.cam_x,world.cam_y,world.cam_z)
                if dist <= range_studs then
                    local sx, sy, vis = draw.world_to_screen(pos.x, pos.y, pos.z)
                    if vis then
                        local dm = floor_calc(dist * M_PER_STUDS)
                        local txt = entry.name .. " [" .. dm .. "m]"
                        local tw = draw.get_text_size(txt, 13)
                        draw.text(sx - tw*0.5, sy, txt, S.container_col, 13)
                        if S.container_contents and entry.contents then
                            local ly = sy + 14
                            for j=1,#entry.contents do
                                local itxt = entry.contents[j]
                                local itw = draw.get_text_size(itxt, 11)
                                draw.text(sx - itw*0.5, ly, itxt, {0.7,0.7,0.7,1}, 11)
                                ly = ly + 12
                            end
                        end
                    end
                end
            end
        end
    end
end

local function draw_quest_esp()
    if not S.quest or #quest_points == 0 then return end
    local range_studs = S.quest_range_studs
    for i=1,#quest_points do
        local entry = quest_points[i]
        local part = entry.part
        if part then
            local pos = part.position
            if pos then
                local dist = dist3(pos.x,pos.y,pos.z, world.cam_x,world.cam_y,world.cam_z)
                if dist <= range_studs then
                    local sx, sy, vis = draw.world_to_screen(pos.x, pos.y, pos.z)
                    if vis then
                        local dm = floor_calc(dist * M_PER_STUDS)
                        local txt = entry.name .. " [" .. dm .. "m]"
                        local tw = draw.get_text_size(txt, 13)
                        draw.text(sx - tw*0.5, sy, txt, S.quest_col, 13)
                    end
                end
            end
        end
    end
end

local function draw_claymore_esp()
    if not S.claymore or #claymore_points == 0 then return end
    local range_studs = S.claymore_range_studs
    for i=1,#claymore_points do
        local entry = claymore_points[i]
        local part = entry.part
        if part then
            local pos = part.position
            if pos then
                local dist = dist3(pos.x,pos.y,pos.z, world.cam_x,world.cam_y,world.cam_z)
                if dist <= range_studs then
                    local sz = part.size
                    local hx = sz and sz.x*0.5 or 1
                    local hy = sz and sz.y*0.5 or 1
                    local hz = sz and sz.z*0.5 or 1
                    local minx,miny,maxx,maxy = 10000,10000,-10000,-10000
                    local valid = false
                    for ox=-1,1,2 do for oy=-1,1,2 do for oz=-1,1,2 do
                        local sx,sy,vis = draw.world_to_screen(pos.x+hx*ox, pos.y+hy*oy, pos.z+hz*oz)
                        if vis then
                            valid = true
                            if sx<minx then minx=sx end if sx>maxx then maxx=sx end
                            if sy<miny then miny=sy end if sy>maxy then maxy=sy end
                        end
                    end end end
                    if valid then
                        local bw = maxx-minx
                        local bh = maxy-miny
                        draw.box(minx, miny, bw, bh, S.claymore_col, 1)
                        if S.claymore_names then
                            local dm = floor_calc(dist * M_PER_STUDS)
                            local txt = entry.name .. " [" .. dm .. "m]"
                            local tw = draw.get_text_size(txt, 13)
                            draw.text(minx + (bw - tw) * 0.5, miny - 16, txt, S.claymore_col, 13)
                        end
                    end
                end
            end
        end
    end
end

local BOSS_HUD_POS = { x = 20, y = 200 }
local BOSS_HUD_DRAG = { dragging = false, ox = 0, oy = 0 }

local function format_boss_line(entry)
    if not entry.found then return "UNSPAWNED", {0.40,0.40,0.45,1} end
    if entry.kind == "vehicle" then
        local dist_txt = "--"
        if entry.hrp and entry.hrp.position then
            local p = entry.hrp.position
            local d = dist3(p.x,p.y,p.z, world.cam_x,world.cam_y,world.cam_z)
            dist_txt = floor_calc(d * M_PER_STUDS) .. "m"
        end
        return dist_txt, {1, 0.85, 0.20, 1}
    end
    if not entry.alive then return "[DEAD]", {0.45,0.45,0.50,1} end
    local dist_txt = "--"
    if entry.hrp and entry.hrp.position then
        local p = entry.hrp.position
        local d = dist3(p.x,p.y,p.z, world.cam_x,world.cam_y,world.cam_z)
        dist_txt = floor_calc(d * M_PER_STUDS) .. "m"
    end
    local hp_txt = "HP " .. floor_calc(entry.hp) .. "/" .. floor_calc(entry.max_hp)
    return "[" .. dist_txt .. "]  " .. hp_txt, {1, 0.30, 0.30, 1}
end

local function draw_boss_tracker()
    if not S.boss then BOSS_HUD_DRAG.dragging = false return end
    local TITLE_H = 22
    local SEC_H = 18
    local LINE_H = 20
    local PAD = 8
    local FONT = 13
    local FONT_TITLE = 14
    local FONT_SEC = 10
    local W = 290
    local est_rows = #BOSS_TARGETS_ESTONIA
    local city_rows = #BOSS_TARGETS_CITY13
    local H = TITLE_H + PAD + SEC_H + (est_rows * LINE_H) + 6
        + SEC_H + (city_rows * LINE_H) + 6 + PAD
    local mx, my = utility.get_mouse_pos()
    local lmb = input.is_key_down(0x01)
    local x, y = BOSS_HUD_POS.x, BOSS_HUD_POS.y
    if lmb then
        if not BOSS_HUD_DRAG.dragging then
            if mx >= x and mx <= x + W and my >= y and my <= y + TITLE_H then
                BOSS_HUD_DRAG.dragging = true
                BOSS_HUD_DRAG.ox = mx - x
                BOSS_HUD_DRAG.oy = my - y
            end
        else
            x = mx - BOSS_HUD_DRAG.ox
            y = my - BOSS_HUD_DRAG.oy
            BOSS_HUD_POS.x = x
            BOSS_HUD_POS.y = y
        end
    else
        BOSS_HUD_DRAG.dragging = false
    end
    draw.rect_filled(x, y, W, H, {0.03,0.03,0.05,0.88}, 4)
    draw.rect(x, y, W, H, {0.18,0.72,0.84,0.85}, 4, 1.5)
    draw.rect_filled(x, y, W, TITLE_H, {0.07,0.07,0.10,1}, 4)
    draw.rect_filled(x, y, 2, TITLE_H, {0.18,0.72,0.84,1}, 0)
    draw.line(x, y+TITLE_H, x+W, y+TITLE_H, {0.18,0.72,0.84,0.5}, 1)
    local ttw = draw.get_text_size("BOSS TRACKER", FONT_TITLE)
    draw.text(x + (W - ttw) * 0.5, y + 4, "BOSS TRACKER", {0.18,0.72,0.84,1}, FONT_TITLE)
    local ly = y + TITLE_H + PAD
    local function sec_header(label)
        draw.rect_filled(x, ly, W, SEC_H, {0.08,0.08,0.11,1}, 0)
        draw.rect_filled(x, ly, 2, SEC_H, {0.18,0.72,0.84,1}, 0)
        draw.line(x, ly + SEC_H, x + W, ly + SEC_H, {0.18,0.72,0.84,0.4}, 1)
        local tw = draw.get_text_size(label, FONT_SEC)
        draw.text(x + (W - tw) * 0.5, ly + 3, label, {0.18,0.72,0.84,1}, FONT_SEC)
        ly = ly + SEC_H
    end
    local function row(entry)
        draw.text(x + PAD, ly, entry.label, {0.95,0.95,0.95,1}, FONT)
        local txt, col = format_boss_line(entry)
        local tw = draw.get_text_size(txt, FONT)
        draw.text(x + W - PAD - tw, ly, txt, col, FONT)
        ly = ly + LINE_H
    end
    local est = boss_cached.estonia or {}
    local city = boss_cached.city13 or {}
    sec_header("ESTONIA")
    for _, boss in ipairs(BOSS_TARGETS_ESTONIA) do
        local e = est[boss.key]
        if e then row(e) end
    end
    ly = ly + 6
    sec_header("CITY-13")
    for _, boss in ipairs(BOSS_TARGETS_CITY13) do
        local e = city[boss.key]
        if e then row(e) end
    end
end

local lt_panel = { x = 30, y = 700, w = 280, dragging = false, drag_ox = 0, drag_oy = 0 }
local LT_TITLE_H = 22
local LT_PAD     = 8
local LT_LINE_H  = 16
local LT_FONT    = 13
local LT_FONT_S  = 11

local lt_col_bg      = { 0.03, 0.03, 0.05, 0.88 }
local lt_col_title   = { 0.07, 0.07, 0.10, 1.00 }
local lt_col_border  = { 0.18, 0.72, 0.84, 0.85 }
local lt_col_accent  = { 0.18, 0.72, 0.84, 1.00 }
local lt_col_player  = { 1.00, 1.00, 1.00, 1.00 }
local lt_col_item    = { 0.35, 0.90, 0.45, 1.00 }
local lt_col_none    = { 0.45, 0.45, 0.50, 1.00 }
local lt_col_sep     = { 0.18, 0.72, 0.84, 0.30 }
local lt_col_far     = { 1.00, 0.60, 0.20, 1.00 }

local LT_TRACKED_ITEMS = {
    "GPU", "Gold50g", "SPSh44", "FlareGun", "Mag556Rnd100",
    "SOCOM556", "Reapir", "HSPV", "TitanShield", "GoldenTicket",
    "OONTAIN", "CatToken", "GoldSkull", "WeddingRing",
    "ThermalImagerCore", "Thermal Imager Core", "Thermal_Imager_Core",
    "MPSU", "Military Power Supply Unit",
    "M4", "M4A1", "AsVal", "R700", "TFZ98",
    "SolterStatue", "CPU", "RepairKit",
}

local LT_LOOKUP = {}
for _, it in ipairs(LT_TRACKED_ITEMS) do LT_LOOKUP[it:lower()] = true end

local cached_players = {}
local rs_players_cache = nil
local rs_players_cache_last = 0

local function get_rs_players()
    local now = utility.get_tick_count()
    if rs_players_cache and (now - rs_players_cache_last) < 5000 and is_valid(rs_players_cache) then
        return rs_players_cache
    end
    local rep = game.replicated_storage
    rs_players_cache = rep and rep:find_first_child("Players") or nil
    rs_players_cache_last = now
    return rs_players_cache
end

local function item_is_tracked(item)
    if type(item) ~= "string" or item == "" then return false end
    local il = item:lower()
    if is_weapon_attachment(item) then return false end
    if LT_LOOKUP[il] then return true end
    if il == "m4a1" and LT_LOOKUP["m4"] then return true end
    return false
end

local function get_weapon_attachments(player_obj)
    if not player_obj then return {} end
    local ok, char = pcall(function() return player_obj.character end)
    if not ok or not char then return {} end
    local ok2, holding = pcall(function() return char:find_first_child("Holding") end)
    if not ok2 or not holding then return {} end
    local ok3, weapon = pcall(function() return holding.value end)
    if not ok3 or not weapon then return {} end
    local attachments = {}
    local ok4, kids = pcall(function() return weapon:get_children() end)
    if ok4 and kids then
        for i = 1, #kids do
            local k = kids[i]
            if k then
                local n = pcall(function() return k.name end) and k.name or ""
                if n ~= "" and item_is_tracked(n) then attachments[#attachments+1] = n end
                local ok5, sub = pcall(function() return k:get_children() end)
                if ok5 and sub then
                    for j = 1, #sub do
                        local s = sub[j]
                        if s then
                            local sn = pcall(function() return s.name end) and s.name or ""
                            if sn ~= "" and item_is_tracked(sn) then attachments[#attachments+1] = sn end
                        end
                    end
                end
            end
        end
    end
    return attachments
end

local function scan_players_inv()
    if not S.lt then cached_players = {} return end
    local result = {}
    local players = entity.get_players()
    if not players then cached_players = result return end
    local rep_players = get_rs_players()
    for i = 1, #players do
        local p = players[i]
        if p and p.is_valid and not p.is_local then
            local pname = p.name or ""
            local puid  = tostring(p.user_id or "")
            local inv = {}
            local seen = {}
            local equipped = nil
            local count = 0
            local function add(name)
                if count >= 40 then return end
                if name and name ~= "" and not seen[name] then
                    seen[name] = true
                    inv[#inv+1] = name
                    count = count + 1
                end
            end
            local char = p.character
            if char then
                local holding = char:find_first_child("Holding")
                if holding then
                    local ok, val = pcall(function() return holding.value end)
                    if ok and val then
                        local ok2, nm = pcall(function() return val.name end)
                        if ok2 and nm and nm ~= "" then
                            equipped = nm
                            add(nm)
                        end
                    end
                end
                local attachments = get_weapon_attachments(p)
                for k = 1, #attachments do add(attachments[k]) end
            end
            if rep_players then
                local data = rep_players:find_first_child(pname)
                if not data and puid ~= "" then data = rep_players:find_first_child(puid) end
                if data then
                    local inv_folder = data:find_first_child("Inventory")
                    if inv_folder then
                        local ok, kids = pcall(function() return inv_folder:get_children() end)
                        if ok and kids then
                            for j = 1, #kids do
                                if count >= 40 then break end
                                local it = kids[j]
                                if it then
                                    local nm = it.name
                                    if type(nm) == "string" and nm ~= "" then
                                        local resolved = resolve_item_name(it)
                                        if resolved then add(resolved) else add(nm) end
                                    end
                                end
                            end
                        end
                    end
                end
            end
            result[#result+1] = { name = pname, entity = p, inventory = inv, equipped = equipped }
        end
    end
    cached_players = result
end

local lt_results = {}

local function lt_scan()
    if not S.lt then lt_results = {} return end
    local found = {}
    local far_threshold = S.lt_far_studs
    for i = 1, #cached_players do
        local d = cached_players[i]
        local p = d.entity
        if p and p.is_valid and p.is_alive and not p.is_local then
            local matched = {}
            local seen = {}
            for _, item in ipairs(d.inventory) do
                if type(item) == "string" and item ~= "" and not seen[item] then
                    if item_is_tracked(item) then
                        seen[item] = true
                        matched[#matched+1] = item
                    end
                end
            end
            if type(d.equipped) == "string" and d.equipped ~= "" and not seen[d.equipped] then
                if item_is_tracked(d.equipped) then
                    seen[d.equipped] = true
                    matched[#matched+1] = d.equipped
                end
            end
            if #matched > 0 then
                local pos = p.position
                local dist = 99999
                if pos then dist = dist3(pos.x, pos.y, pos.z, world.cam_x, world.cam_y, world.cam_z) end
                local too_far = dist > far_threshold
                found[#found+1] = {name = d.name, items = matched, dist = dist, too_far = too_far}
            end
        end
    end
    table.sort(found, function(a, b) return a.dist < b.dist end)
    local limited = {}
    local count = 0
    local max_entries = S.lt_max or 50
    for _, entry in ipairs(found) do
        if count >= max_entries then break end
        local remaining = max_entries - count
        local taken = {}
        for _, it in ipairs(entry.items) do
            if #taken >= remaining then break end
            taken[#taken+1] = it
        end
        if #taken > 0 then
            limited[#limited+1] = {name = entry.name, items = taken, dist = entry.dist, too_far = entry.too_far}
            count = count + #taken
        end
    end
    lt_results = limited
end

local function draw_loot_tracker()
    if not S.lt then lt_panel.dragging = false return end
    local results = lt_results
    local num_results = #results
    local panel_h = LT_TITLE_H + LT_PAD
    if num_results == 0 then
        panel_h = panel_h + LT_LINE_H + LT_PAD
    else
        for _, entry in ipairs(results) do
            panel_h = panel_h + LT_LINE_H
            panel_h = panel_h + (#entry.items * LT_LINE_H)
            panel_h = panel_h + 6
        end
        panel_h = panel_h + LT_PAD
    end
    local pw = lt_panel.w
    local mx, my = utility.get_mouse_pos()
    local lmb = input.is_key_down(0x01)
    if lmb then
        if not lt_panel.dragging then
            if mx >= lt_panel.x and mx <= lt_panel.x + pw and my >= lt_panel.y and my <= lt_panel.y + LT_TITLE_H then
                lt_panel.dragging = true
                lt_panel.drag_ox = mx - lt_panel.x
                lt_panel.drag_oy = my - lt_panel.y
            end
        else
            lt_panel.x = mx - lt_panel.drag_ox
            lt_panel.y = my - lt_panel.drag_oy
        end
    else
        lt_panel.dragging = false
    end
    local px, py = lt_panel.x, lt_panel.y
    draw.rect(px-1, py-1, pw+2, panel_h+2, {0.18,0.72,0.84,0.15}, 5, 1)
    draw.rect_filled(px, py, pw, panel_h, lt_col_bg, 4)
    draw.rect_filled(px, py, pw, LT_TITLE_H, lt_col_title, 4)
    draw.rect_filled(px, py, 2, LT_TITLE_H, lt_col_border, 0)
    draw.line(px, py+LT_TITLE_H, px+pw, py+LT_TITLE_H, lt_col_border, 1)
    draw.rect(px, py, pw, panel_h, lt_col_border, 4, 1)
    local title = "LOOT TRACKER"
    if num_results > 0 then title = title .. "  [" .. num_results .. "]" end
    local ttw = draw.get_text_size(title, LT_FONT)
    draw.text(px + (pw - ttw) * 0.5, py + 4, title, lt_col_accent, LT_FONT)
    local ly = py + LT_TITLE_H + LT_PAD
    if num_results == 0 then
        draw.text(px + LT_PAD, ly, "No tracked items found", lt_col_none, LT_FONT)
        return
    end
    for _, entry in ipairs(results) do
        local dist_txt, dist_col
        if entry.too_far then
            dist_txt = "[TOO FAR]"
            dist_col = lt_col_far
        else
            local dm = floor_calc(entry.dist * M_PER_STUDS)
            dist_txt = "[" .. dm .. "m]"
            dist_col = lt_col_player
        end
        local name_txt = "> " .. entry.name .. "  " .. dist_txt
        draw.text(px + LT_PAD, ly, name_txt, dist_col, LT_FONT)
        ly = ly + LT_LINE_H
        for _, item in ipairs(entry.items) do
            draw.text(px + LT_PAD + 8, ly, "- " .. item, lt_col_item, LT_FONT_S)
            ly = ly + LT_LINE_H
        end
        draw.line(px + LT_PAD, ly + 1, px + pw - LT_PAD, ly + 1, lt_col_sep, 1)
        ly = ly + 6
    end
end

local aim_target_ent = nil
local locked_target = nil
local lock_was_down = false

local function target_still_valid(t)
    if not t then return false end
    if t.kind == "player" then
        local p = t.entity
        if not p or not p.is_valid or not p.is_alive then return false end
        local pos = p.position
        if not pos then return false end
        return true
    elseif t.kind == "npc" then
        local n = t.entity
        if not n or not n.hum or not n.hum.health or n.hum.health <= 0 then return false end
        if not n.head or not n.head.position then return false end
        return true
    end
    return false
end

local function target_in_fov(t, scx, scy)
    if not t then return false end
    local bx, by, bvis
    if t.kind == "player" then
        local p = t.entity
        if not p then return false end
        if t.bone_name == "Head" and p.head_position then
            bx, by, bvis = draw.world_to_screen(p.head_position.x, p.head_position.y, p.head_position.z)
        else
            bx, by, bvis = p:GetBoneScreen(t.bone_name)
        end
    elseif t.kind == "npc" then
        local n = t.entity
        if not n or not n.parts or not n.parts[t.bone_name] or not n.parts[t.bone_name].part then return false end
        local pos = n.parts[t.bone_name].part.position
        if not pos then return false end
        bx, by, bvis = draw.world_to_screen(pos.x, pos.y, pos.z)
    end
    if not bvis then return false end
    local dx, dy = bx - scx, by - scy
    return sqrt(dx*dx + dy*dy) <= S.aim_fov
end

local function pick_target(scx, scy)
    local best_fov = S.aim_fov
    local best = nil
    local cx,cy,cz = world.cam_x, world.cam_y, world.cam_z
    local bone_map = {"Head","UpperTorso","LowerTorso"}
    local bone_idx = tonumber(S.aim_bone) or 0
    if bone_idx < 0 or bone_idx > 2 then bone_idx = 0 end
    local bone_name = bone_map[bone_idx + 1] or "Head"
    local players = cached_players_frame
    local lp = nil
    local local_team = nil
    if #players > 0 then
        lp = entity.get_local_player()
        local_team = lp and lp.team
    end
    if S.aim_target ~= 2 then
        for i=1,#players do
            local p = players[i]
            if p and p.is_valid and p.is_alive and not p.is_local then
                local skip = false
                if S.player_team and p.has_team and local_team and p.team == local_team then skip = true end
                if not skip then
                    local bx, by, bvis
                    if bone_name == "Head" and p.head_position then
                        bx, by, bvis = draw.world_to_screen(p.head_position.x, p.head_position.y, p.head_position.z)
                    else
                        bx, by, bvis = p:GetBoneScreen(bone_name)
                    end
                    if bvis then
                        local dx, dy = bx - scx, by - scy
                        local fov = sqrt(dx*dx + dy*dy)
                        if fov < best_fov then
                            if not S.aim_visible or raycast.IsPlayerVisible(p.character) then
                                best_fov = fov
                                best = {kind="player", entity=p, bone_name=bone_name, velocity=p.velocity, position=p.position}
                            end
                        end
                    end
                end
            end
        end
    end
    if S.aim_target ~= 1 then
        for i=1,#npcs_cached do
            local n = npcs_cached[i]
            if n.hum and n.hum.health and n.hum.health > 0 then
                local pd = n.parts[bone_name]
                if pd and pd.part then
                    local pos = pd.part.position
                    if pos then
                        local sx, sy, vis = draw.world_to_screen(pos.x, pos.y, pos.z)
                        if vis then
                            local dx, dy = sx - scx, sy - scy
                            local fov = sqrt(dx*dx + dy*dy)
                            if fov < best_fov then
                                if not S.aim_visible or raycast.IsVisible(cx, cy, cz, pos.x, pos.y, pos.z) then
                                    best_fov = fov
                                    best = {kind="npc", entity=n, bone_name=bone_name, velocity=pd.part.velocity, position=pos}
                                end
                            end
                        end
                    end
                end
            end
        end
    end
    return best
end

local function refresh_locked_target_pos()
    if not locked_target then return end
    local t = locked_target
    if t.kind == "player" then
        local p = t.entity
        if p and p.is_valid and p.is_alive and p.position then
            t.position = p.position
            t.velocity = p.velocity
        end
    elseif t.kind == "npc" then
        local n = t.entity
        if n and n.parts and n.parts[t.bone_name] and n.parts[t.bone_name].part then
            local part = n.parts[t.bone_name].part
            if part.position then
                t.position = part.position
                t.velocity = part.velocity
            end
        end
    end
end

local function compute_aim_point(t)
    if not t or not t.position then return nil end
    local px, py, pz
    if t.kind == "player" and t.bone_name == "Head" then
        local p = t.entity
        if p and p.head_position then
            px, py, pz = p.head_position.x, p.head_position.y, p.head_position.z
        else
            px, py, pz = t.position.x, t.position.y, t.position.z
        end
    else
        px, py, pz = t.position.x, t.position.y, t.position.z
    end
    local lp = entity.get_local_player()
    if not lp or not lp.head_position then return px, py, pz end
    local cam = lp.head_position
    local dx_s = px - cam.x
    local dy_s = py - cam.y
    local dz_s = pz - cam.z
    local dist_studs = sqrt(dx_s*dx_s + dy_s*dy_s + dz_s*dz_s)
    local dist_m = dist_studs * M_PER_STUDS
    local drop_m = 0
    local t_flight = 0
    if S.aim_predict then
        local weapon = get_equipped_weapon(lp)
        local speed_ms = get_bullet_speed(weapon)
        local d_m, tf = calc_ballistic_drop(dist_m, speed_ms, GRAVITY_MS2)
        drop_m = d_m * S.aim_predict_scale
        t_flight = tf
    end
    local lead_x_m, lead_y_m, lead_z_m = 0, 0, 0
    if S.aim_lead and t.velocity then
        local v = t.velocity
        local tf = t_flight
        if tf <= 0 then
            local weapon = get_equipped_weapon(lp)
            local speed_ms = get_bullet_speed(weapon)
            local _, calc_t = calc_ballistic_drop(dist_m, speed_ms, GRAVITY_MS2)
            tf = calc_t
        end
        lead_x_m = (v.x * M_PER_STUDS) * tf
        lead_y_m = (v.y * M_PER_STUDS) * tf
        lead_z_m = (v.z * M_PER_STUDS) * tf
    end
    local aim_x_m = (px * M_PER_STUDS) + lead_x_m
    local aim_y_m = (py * M_PER_STUDS) + lead_y_m + drop_m
    local aim_z_m = (pz * M_PER_STUDS) + lead_z_m
    return aim_x_m * STUDS_PER_M, aim_y_m * STUDS_PER_M, aim_z_m * STUDS_PER_M
end

local function run_aimbot()
    aim_target_ent = nil
    if not S.aim then
        locked_target = nil
        lock_was_down = false
        return
    end
    local scx, scy = input.get_screen_center()
    if S.aim_draw_fov then
        draw.circle(scx, scy, S.aim_fov, {1,1,1,0.35}, 64, 1)
    end
    local key_down_aim = input.is_key_down(S.aim_key)
    if not key_down_aim then
        locked_target = nil
        lock_was_down = false
        return
    end
    if S.aim_lock then
        if not lock_was_down then
            locked_target = pick_target(scx, scy)
            lock_was_down = true
        end
        if locked_target and not target_still_valid(locked_target) then locked_target = nil end
        if locked_target and not target_in_fov(locked_target, scx, scy) then locked_target = nil end
        if not locked_target then locked_target = pick_target(scx, scy) end
        if locked_target then refresh_locked_target_pos() end
        local target = locked_target
        if not target then return end
        aim_target_ent = target
        local ax, ay, az = compute_aim_point(target)
        if ax and ay and az then camera.look_at(ax, ay, az, S.aim_smooth) end
    else
        locked_target = nil
        lock_was_down = false
        local target = pick_target(scx, scy)
        if not target then return end
        aim_target_ent = target
        local ax, ay, az = compute_aim_point(target)
        if ax and ay and az then camera.look_at(ax, ay, az, S.aim_smooth) end
    end
end

local INVENTORY_CACHE = {}
local inv_panel = { x = 1660, y = 260, w = 320, dragging = false, drag_ox = 0, drag_oy = 0 }
local inv_toggled = false
local inv_last_key = false

local function scan_player_inventories()
    if not S.inv then INVENTORY_CACHE = {} return end
    local result = {}
    local players = entity.get_players()
    if not players then return end
    local rep_players = get_rs_players()
    for i = 1, #players do
        local p = players[i]
        if p and p.is_valid and not p.is_local then
            local pname = p.name or ""
            local puid = tostring(p.user_id or "")
            local items = {}
            local seen = {}
            local function add(name)
                if name and name ~= "" and not seen[name] then
                    seen[name] = true
                    items[#items+1] = name                end
            end
            if rep_players then
                local data = rep_players:find_first_child(pname)
                if not data and puid ~= "" then data = rep_players:find_first_child(puid) end
                if data then
                    local inv_f = data:find_first_child("Inventory")
                    if inv_f then
                        local ok, kids = pcall(function() return inv_f:get_children() end)
                        if ok and kids then
                            for j = 1, #kids do
                                local it = kids[j]
                                if it then
                                    local nm = it.name
                                    if type(nm) == "string" and nm ~= "" then
                                        local resolved = resolve_item_name(it)
                                        if resolved then add(resolved) else add(nm) end
                                    end
                                end
                            end
                        end
                    end
                    local vault_f = data:find_first_child("VaultStorage")
                    if vault_f then
                        local ok, kids = pcall(function() return vault_f:get_children() end)
                        if ok and kids then
                            for j = 1, #kids do
                                local it = kids[j]
                                if it then
                                    local nm = it.name
                                    if type(nm) == "string" and nm ~= "" then
                                        local resolved = resolve_item_name(it)
                                        if resolved then add(resolved) else add(nm) end
                                    end
                                end
                            end
                        end
                    end
                end
            end
            local char = p.character
            if char then
                local clothing = char:find_first_child("Clothing")
                if clothing then
                    local ok, kids = pcall(function() return clothing:get_children() end)
                    if ok and kids then
                        for j=1,#kids do
                            local slot = kids[j]
                            if slot then
                                local nm = slot.name
                                if type(nm) == "string" and nm ~= "" then
                                    local resolved = resolve_item_name(slot)
                                    if resolved then add(resolved) end
                                end
                            end
                        end
                    end
                end
                local ok, ch = pcall(function() return char:get_children() end)
                if ok and ch then
                    for j=1,#ch do
                        local c = ch[j]
                        if c and c.class_name == "Model" then
                            local cn = c.name or ""
                            if cn ~= "" and not cn:find("Holstered") and not cn:find("Wasteland") and not cn:find("Summer") and not cn:find("Civilian") then
                                cn = ITEM_NAME_MAP[cn] or cn
                                add(cn)
                            end
                        elseif c and c.class_name == "Accessory" then
                            local n = resolve_item_name(c)
                            if n then add(n) end
                        end
                    end
                end
                local attachments = get_weapon_attachments(p)
                for k = 1, #attachments do add(attachments[k]) end
            end
            result[pname] = {items = items, entity = p, equipped = get_equipped_weapon(p)}
        end
    end
    INVENTORY_CACHE = result
end

local function get_closest_visible_player()
    local sw, sh = draw.get_screen_size()
    local scx, scy = sw * 0.5, sh * 0.5
    local best, best_dist = nil, math.huge
    for pname, data in pairs(INVENTORY_CACHE) do
        local p = data.entity
        if p and p.is_valid and p.is_alive and p.position then
            local pos = p.position
            local sx, sy, vis = draw.world_to_screen(pos.x, pos.y, pos.z)
            if vis then
                local dx, dy = sx - scx, sy - scy
                local dist = sqrt(dx*dx + dy*dy)
                if dist < best_dist then best_dist = dist best = data end
            end
        end
    end
    return best
end

local function get_crosshair_player()
    local sw, sh = draw.get_screen_size()
    local scx, scy = sw * 0.5, sh * 0.5
    local best, best_dist = nil, math.huge
    local players = cached_players_frame
    if #players == 0 then return nil end
    local range_studs = S.hud_range_studs
    for i=1,#players do
        local p = players[i]
        if p and p.is_valid and p.is_alive and not p.is_local then
            local pos = p.position
            if pos then
                local dist = dist3(pos.x,pos.y,pos.z, world.cam_x,world.cam_y,world.cam_z)
                if dist <= range_studs then
                    local sx, sy, vis = draw.world_to_screen(pos.x, pos.y, pos.z)
                    if vis then
                        local dx, dy = sx - scx, sy - scy
                        local screen_dist = sqrt(dx*dx + dy*dy)
                        if screen_dist < best_dist then
                            best_dist = screen_dist
                            best = p
                        end
                    end
                end
            end
        end
    end
    return best
end

local function should_show_inv()
    if not S.inv then return false end
    if S.inv_mode == 0 then return true end
    if S.inv_mode == 1 then
        local down = input.is_key_down(S.inv_key)
        if down and not inv_last_key then inv_toggled = not inv_toggled end
        inv_last_key = down
        return inv_toggled
    end
    inv_last_key = false
    return input.is_key_down(S.inv_key)
end

local function draw_section_header(px, ly, pw, label, col_left)
    local SEC_H = 18
    draw.rect_filled(px, ly, pw, SEC_H, {0.08,0.08,0.11,1}, 0)
    draw.rect_filled(px, ly, 2, SEC_H, col_left, 0)
    draw.line(px, ly + SEC_H, px + pw, ly + SEC_H, {0.18,0.72,0.84,0.4}, 1)
    local tw = draw.get_text_size(label, 10)
    draw.text(px + (pw - tw) * 0.5, ly + 4, label, col_left, 10)
    return ly + SEC_H
end

local function clamp_list(list, mx)
    if #list <= mx then return list end
    local out = {}
    for i = 1, mx do out[i] = list[i] end
    return out
end

local function draw_inventory_panel()
    if not should_show_inv() then inv_panel.dragging = false return end
    local target = get_closest_visible_player()
    if not target then return end
    local weapons_t, armor_t, valuable_t, other_t = {}, {}, {}, {}
    for _, item in ipairs(target.items) do
        if not is_weapon_attachment(item) then
            local cat = categorize(item)
            if cat == "weapon" then
                weapons_t[#weapons_t+1] = item
            elseif cat == "armor" then
                armor_t[#armor_t+1] = item
            elseif cat == "valuable" then
                valuable_t[#valuable_t+1] = item
            elseif item_is_tracked(item) then
                valuable_t[#valuable_t+1] = item
            else
                other_t[#other_t+1] = item
            end
        end
    end
    weapons_t  = clamp_list(weapons_t,  S.inv_max_guns)
    armor_t    = clamp_list(armor_t,    S.inv_max_armor)
    valuable_t = clamp_list(valuable_t, S.inv_max_val)
    other_t    = clamp_list(other_t,    S.inv_max_other)
    local TITLE_H = 24
    local PAD = 10
    local LINE_H = 16
    local LINE_H_L = 22
    local FONT = 13
    local FONT_L = 14
    local ICON_SZ = S.inv_icons and S.inv_icon_size or 0
    local TEXT_OFFSET = ICON_SZ > 0 and (ICON_SZ + 4) or 6
    local panel_h = TITLE_H + PAD
    if S.inv_equipped and target.equipped then panel_h = panel_h + LINE_H_L + PAD + 14 end
    if S.inv_guns and #weapons_t > 0 then panel_h = panel_h + 18 + (#weapons_t * LINE_H_L) + 4 end
    if S.inv_armor and #armor_t > 0 then panel_h = panel_h + 18 + (#armor_t * LINE_H_L) + 4 end
    if S.inv_valuables and #valuable_t > 0 then panel_h = panel_h + 18 + (#valuable_t * LINE_H_L) + 4 end
    if S.inv_other and #other_t > 0 then panel_h = panel_h + 18 + (#other_t * LINE_H) + 4 end
    panel_h = panel_h + PAD
    local pw = inv_panel.w
    local mx, my = utility.get_mouse_pos()
    local lmb = input.is_key_down(0x01)
    if lmb then
        if not inv_panel.dragging then
            if mx >= inv_panel.x and mx <= inv_panel.x + pw and my >= inv_panel.y and my <= inv_panel.y + TITLE_H then
                inv_panel.dragging = true
                inv_panel.drag_ox = mx - inv_panel.x
                inv_panel.drag_oy = my - inv_panel.y
            end
        else
            inv_panel.x = mx - inv_panel.drag_ox
            inv_panel.y = my - inv_panel.drag_oy
        end
    else inv_panel.dragging = false end
    local px, py = inv_panel.x, inv_panel.y
    draw.rect(px-1, py-1, pw+2, panel_h+2, {0.18,0.72,0.84,0.15}, 5, 1)
    draw.rect_filled(px, py, pw, panel_h, {0.05,0.05,0.07,0.96}, 4)
    draw.rect_filled(px, py, pw, TITLE_H, {0.07,0.07,0.10,1}, 4)
    draw.rect_filled(px, py, 2, TITLE_H, {0.18,0.72,0.84,1}, 0)
    draw.line(px, py+TITLE_H, px+pw, py+TITLE_H, {0.18,0.72,0.84,1}, 1)
    draw.rect(px, py, pw, panel_h, {0.18,0.72,0.84,0.5}, 4, 1)
    draw.text(px+10, py+5, target.entity.name or "?", {0.18,0.72,0.84,1}, FONT)
    local hint = "drag ::"
    local hw = draw.get_text_size(hint, 10)
    draw.text(px+pw-hw-PAD, py+6, hint, {0.25,0.25,0.30,1}, 10)
    local ly = py + TITLE_H + PAD
    if S.inv_equipped and target.equipped then
        local cat_eq = categorize(target.equipped)
        local col = loot_colors[cat_eq] or loot_colors.other
        draw.text(px+PAD, ly, "EQUIPPED", {0.25,0.25,0.30,1}, 9)
        local wy = ly + 11
        if ICON_SZ > 0 then
            local icon = get_icon_handle(target.equipped)
            if icon then draw.Image(icon, px+PAD, wy, ICON_SZ, ICON_SZ) end
        end
        draw.text(px+PAD+TEXT_OFFSET, wy+2, target.equipped, col, FONT_L)
        ly = ly + LINE_H_L + PAD
        draw.line(px+PAD, ly, px+pw-PAD, ly, {0.18,0.72,0.84,0.4}, 1)
        ly = ly + 4
    end
    if S.inv_guns and #weapons_t > 0 then
        ly = draw_section_header(px, ly, pw, "GUN(S)", {0.95,0.30,0.30,1})
        for _, item in ipairs(weapons_t) do
            if ICON_SZ > 0 then
                local icon = get_icon_handle(item)
                if icon then draw.Image(icon, px+PAD+6, ly+2, ICON_SZ, ICON_SZ) end
            end
            draw.text(px+PAD+6+TEXT_OFFSET, ly+4, item, loot_colors.weapon, FONT_L)
            ly = ly + LINE_H_L
        end
        ly = ly + 4
    end
    if S.inv_armor and #armor_t > 0 then
        ly = draw_section_header(px, ly, pw, "ARMOR", {0.18,0.72,0.84,1})
        for _, item in ipairs(armor_t) do
            if ICON_SZ > 0 then
                local icon = get_icon_handle(item)
                if icon then draw.Image(icon, px+PAD+6, ly+2, ICON_SZ, ICON_SZ) end
            end
            draw.text(px+PAD+6+TEXT_OFFSET, ly+4, item, loot_colors.armor, FONT_L)
            ly = ly + LINE_H_L
        end
        ly = ly + 4
    end
    if S.inv_valuables and #valuable_t > 0 then
        ly = draw_section_header(px, ly, pw, "VALUE", {0.35,0.90,0.45,1})
        for _, item in ipairs(valuable_t) do
            if ICON_SZ > 0 then
                local icon = get_icon_handle(item)
                if icon then draw.Image(icon, px+PAD+6, ly+2, ICON_SZ, ICON_SZ) end
            end
            draw.text(px+PAD+6+TEXT_OFFSET, ly+4, item, loot_colors.valuable, FONT_L)
            ly = ly + LINE_H_L
        end
        ly = ly + 4
    end
    if S.inv_other and #other_t > 0 then
        ly = draw_section_header(px, ly, pw, "OTHER", {0.40,0.40,0.45,1})
        for _, item in ipairs(other_t) do
            draw.text(px+PAD+6, ly+2, item, loot_colors.other, FONT)
            ly = ly + LINE_H
        end
    end
end

local hud_pos = { x = 1580, y = 620, dragging = false, drag_ox = 0, drag_oy = 0 }

local HUD_COLORS = {
    green  = {0.30, 0.95, 0.40, 1},
    blue   = {0.30, 0.65, 1.00, 1},
    white  = {1.00, 1.00, 1.00, 1},
    red    = {1.00, 0.30, 0.30, 1},
    gray   = {0.55, 0.55, 0.60, 1},
}

local HUD_EMPTY = "--"

local function get_equipped_slot(player_obj, slot_name)
    if not player_obj then return nil end
    local ok, char = pcall(function() return player_obj.character end)
    if not ok or not char then return nil end
    local clothing = char:find_first_child("Clothing")
    if not clothing then return nil end
    local slot = clothing:find_first_child(slot_name)
    if not slot then return nil end
    return resolve_item_name(slot)
end

-- =========================================================
-- CHAMS + VISIBILITY (cache + FOV filter)
-- =========================================================
local chams_hook_active = false
local last_chams_style = -1
local chams_vis_cache = {}
local CHAMS_VIS_CACHE_MS = 150
local chams_part_cache = {}

local function get_character_part_addresses(character)
    if not character then return nil end
    local caddr = nil
    pcall(function() caddr = character.address end)
    if not caddr then return nil end
    local cached = chams_part_cache[caddr]
    if cached then return cached end
    local addrs = {}
    local parts = nil
    pcall(function() parts = character:GetDescendantsOfClass("BasePart") end)
    if parts then
        for i = 1, #parts do
            local a = nil
            pcall(function() a = parts[i].address end)
            if a then addrs[a] = true end
        end
    end
    chams_part_cache[caddr] = addrs
    return addrs
end

local function compute_visibility(target_pos, character)
    if not target_pos then return true end
    local hit, _, _, inst = raycast.Cast(
        world.cam_x, world.cam_y, world.cam_z,
        target_pos.x, target_pos.y, target_pos.z
    )
    if not hit or not inst then return true end
    local addr_inst = nil
    pcall(function() addr_inst = inst.address end)
    if not addr_inst then return true end
    local addrs = get_character_part_addresses(character)
    if addrs and addrs[addr_inst] then return true end
    return false
end

local function is_in_vis_fov(pos)
    if not pos then return false end
    local dx = pos.x - world.cam_x
    local dy = pos.y - world.cam_y
    local dz = pos.z - world.cam_z
    local dist = sqrt(dx*dx + dy*dy + dz*dz)
    if dist < 1 then return true end
    local lv = camera.get_look_vector()
    if not lv then return true end
    local nx, ny, nz = dx/dist, dy/dist, dz/dist
    local dot = nx*lv.x + ny*lv.y + nz*lv.z
    local fov = S.chams_vis_fov or 90
    if fov >= 180 then return true end
    local cos_half = math.cos(math.rad(fov * 0.5))
    return dot >= cos_half
end

local function get_cached_visibility(addr, target_pos, character)
    local now = utility.get_tick_count()
    local c = chams_vis_cache[addr]
    if c and (now - c.t) < CHAMS_VIS_CACHE_MS then
        return c.visible
    end
    local visible = compute_visibility(target_pos, character)
    chams_vis_cache[addr] = { visible = visible, t = now }
    return visible
end

-- =========================================================
-- TARGET HUD (com badge VISIBLE/HIDDEN)
-- =========================================================
local function draw_target_hud()
    if not S.hud then return end
    local target = get_crosshair_player()
    if not target then return end
    local name = target.name or "?"
    local weapon = get_equipped_weapon(target)
    local hp = target.health or 0
    local max_hp = target.max_health or 100
    local helmet = get_equipped_slot(target, "ClothingHeadware")
    local armor = get_equipped_slot(target, "ClothingChestRig")
    local mask = get_equipped_slot(target, "ClothingMask")
    local gloves = get_equipped_slot(target, "ClothingGloves")
    local backpack = get_equipped_slot(target, "ClothingBackpack")
    local dist_m = target:DistanceTo() * M_PER_STUDS

    local vis_state = "unknown"
    local vis_color = {0.55, 0.55, 0.60, 1}
    if S.hud_visible then
        local tpos = target.position
        if tpos and target.character then
            local addr = nil
            pcall(function() addr = target.character.address end)
            if addr then
                local visible = get_cached_visibility(addr, tpos, target.character)
                if visible then
                    vis_state = "VISIBLE"
                    vis_color = {0.30, 0.95, 0.40, 1}
                else
                    vis_state = "HIDDEN"
                    vis_color = {1.00, 0.30, 0.30, 1}
                end
            end
        end
    end

    local info_entries = {}
    if S.hud_weapon then
        table.insert(info_entries, { label = "W", name = weapon or HUD_EMPTY, color = weapon and HUD_COLORS.red or HUD_COLORS.gray })
    end
    if S.hud_helmet then
        table.insert(info_entries, { label = "H", name = helmet or HUD_EMPTY, color = helmet and HUD_COLORS.green or HUD_COLORS.gray })
    end
    if S.hud_armor then
        table.insert(info_entries, { label = "A", name = armor or HUD_EMPTY, color = armor and HUD_COLORS.green or HUD_COLORS.gray })
    end
    if S.hud_mask then
        table.insert(info_entries, { label = "M", name = mask or HUD_EMPTY, color = mask and HUD_COLORS.blue or HUD_COLORS.gray })
    end
    if S.hud_gloves then
        table.insert(info_entries, { label = "G", name = gloves or HUD_EMPTY, color = gloves and HUD_COLORS.blue or HUD_COLORS.gray })
    end
    if S.hud_backpack then
        table.insert(info_entries, { label = "B", name = backpack or HUD_EMPTY, color = backpack and HUD_COLORS.white or HUD_COLORS.gray })
    end
    if S.hud_dist then
        table.insert(info_entries, { label = "", name = floor_calc(dist_m) .. "m", color = HUD_COLORS.gray })
    end

    local ICON_SZ = S.hud_icons and S.hud_icon_size or 0
    local LINE_H = ICON_SZ > 0 and (ICON_SZ + 2) or 14
    local w = 300
    local base_h = 50
    if S.hud_hp then base_h = base_h + 16 end
    local h = base_h + (#info_entries * LINE_H) + 4
    local px = hud_pos.x
    local py = hud_pos.y
    if px == 0 and py == 0 then
        local sw, sh = draw.get_screen_size()
        px = sw * 0.5 - w * 0.5
        py = sh * 0.5 + S.hud_offset_y
        hud_pos.x = px
        hud_pos.y = py
    end
    local mx, my = utility.get_mouse_pos()
    local lmb = input.is_key_down(0x01)
    if lmb then
        if not hud_pos.dragging then
            if mx >= px and mx <= px + w and my >= py and my <= py + 22 then
                hud_pos.dragging = true
                hud_pos.drag_ox = mx - px
                hud_pos.drag_oy = my - py
            end
        else
            px = mx - hud_pos.drag_ox
            py = my - hud_pos.drag_oy
            hud_pos.x = px
            hud_pos.y = py
        end
    else
        hud_pos.dragging = false
    end
    draw.rect_filled(px, py, w, h, {0.03,0.03,0.05,0.88}, 6)
    draw.rect(px, py, w, h, {0.18,0.72,0.84,0.85}, 6, 1.5)
    draw.line(px + 8, py + 22, px + w - 8, py + 22, {0.18,0.72,0.84,0.4}, 1)
    if S.hud_name then
        draw.text(px + 8, py + 4, name, {1, 1, 1, 1}, 15)
    end
    if S.hud_weapon then
        local wtxt = "[" .. (weapon or HUD_EMPTY) .. "]"
        local wtxt_col = weapon and HUD_COLORS.red or HUD_COLORS.gray
        local wtw = draw.get_text_size(wtxt, 13)
        draw.text(px + w - wtw - 8, py + 5, wtxt, wtxt_col, 13)
    end

    if S.hud_visible and vis_state ~= "unknown" then
        local vis_w = draw.get_text_size(vis_state, 12)
        draw.rect_filled(px + w - vis_w - 12, py + 28, vis_w + 8, 14, {0.08,0.08,0.11,0.9}, 3)
        draw.text(px + w - vis_w - 8, py + 30, vis_state, vis_color, 12)
    end

    local ly = py + 28
    if S.hud_hp then
        local pct = max_hp > 0 and hp / max_hp or 0
        local bar_w = 180
        local bar_h = 10
        draw.rect_filled(px + 8, ly, bar_w, bar_h, {0.15,0.15,0.18,1}, 2)
        local hp_col
        if pct > 0.6 then hp_col = {0.3, 0.9, 0.3, 1}
        elseif pct > 0.3 then hp_col = {1, 0.8, 0.2, 1}
        else hp_col = {1, 0.2, 0.2, 1} end
        draw.rect_filled(px + 8, ly, bar_w * pct, bar_h, hp_col, 2)
        local hp_txt = floor_calc(hp) .. "/" .. floor_calc(max_hp)
        draw.text(px + 8 + bar_w + 8, ly - 2, hp_txt, {1, 1, 1, 1}, 13)
        ly = ly + 16
    end
    for i, entry in ipairs(info_entries) do
        local ey = ly + (i - 1) * LINE_H
        local text_x = px + 8
        if ICON_SZ > 0 and entry.name ~= HUD_EMPTY then
            local icon = get_icon_handle(entry.name)
            if icon then
                draw.Image(icon, text_x, ey, ICON_SZ, ICON_SZ)
                text_x = text_x + ICON_SZ + 4
            end
        end
        local prefix = entry.label ~= "" and (entry.label .. ": ") or ""
        draw.text(text_x, ey + (ICON_SZ > 0 and (ICON_SZ/2 - 6) or 0), prefix .. entry.name, entry.color, 12)
    end
end

-- =========================================================
-- CHAMS DRAW
-- =========================================================
local chams_cleanup_counter = 0

local function draw_chams()
    if not S.chams then
        if chams_hook_active then
            pcall(function() exploits.RevertChams() end)
            chams_hook_active = false
        end
        return
    end
    if S.chams_style ~= last_chams_style then
        pcall(function() exploits.SetChamsMode(S.chams_style) end)
        last_chams_style = S.chams_style
        chams_hook_active = true
    end

    local want_visibility = S.chams_gradient == true

    chams_cleanup_counter = chams_cleanup_counter + 1
    if chams_cleanup_counter >= 300 then
        chams_cleanup_counter = 0
        local now = utility.get_tick_count()
        for k, v in pairs(chams_vis_cache) do
            if now - v.t > 2000 then chams_vis_cache[k] = nil end
        end
        if next(chams_part_cache) then
            local keep = {}
            for i = 1, #cached_players_frame do
                local p = cached_players_frame[i]
                if p and p.character then
                    local a = nil
                    pcall(function() a = p.character.address end)
                    if a and chams_part_cache[a] then keep[a] = chams_part_cache[a] end
                end
            end
            for i = 1, #npcs_cached do
                local n = npcs_cached[i]
                if n.model then
                    local a = nil
                    pcall(function() a = n.model.address end)
                    if a and chams_part_cache[a] then keep[a] = chams_part_cache[a] end
                end
            end
            chams_part_cache = keep
        end
    end

    if S.chams_players then
        local players = cached_players_frame
        local range_studs = S.player_range_studs
        for i=1,#players do
            local p = players[i]
            if p and p.is_valid and p.is_alive and not p.is_local then
                local pos = p.position
                if pos then
                    local dist = dist3(pos.x,pos.y,pos.z, world.cam_x,world.cam_y,world.cam_z)
                    if dist <= range_studs then
                        if want_visibility then
                            local in_fov = is_in_vis_fov(pos)
                            if in_fov then
                                local addr = nil
                                pcall(function() addr = p.character and p.character.address end)
                                if addr then
                                    local visible = get_cached_visibility(addr, pos, p.character)
                                    local col = visible and S.chams_col or S.chams_col2
                                    pcall(function() draw.ChamsPlayer(p, col, col, S.chams_style) end)
                                else
                                    pcall(function() draw.ChamsPlayer(p, S.chams_col, S.chams_col, S.chams_style) end)
                                end
                            else
                                pcall(function() draw.ChamsPlayer(p, S.chams_col, S.chams_col, S.chams_style) end)
                            end
                        else
                            pcall(function() draw.ChamsPlayer(p, S.chams_col, S.chams_style) end)
                        end
                    end
                end
            end
        end
    end

    if S.chams_npcs then
        for i=1,#npcs_cached do
            local n = npcs_cached[i]
            if n.model and n.hum and n.hum.health and n.hum.health > 0 then
                local hpos = n.head and n.head.position
                if hpos then
                    local dist = dist3(hpos.x,hpos.y,hpos.z, world.cam_x,world.cam_y,world.cam_z)
                    if dist <= S.npc_range_studs then
                        local hulls = nil
                        pcall(function() hulls = draw.GetPlayerHulls(n.model) end)
                        if hulls then
                            if want_visibility then
                                local in_fov = is_in_vis_fov(hpos)
                                if in_fov then
                                    local addr = nil
                                    pcall(function() addr = n.model.address end)
                                    if addr then
                                        local visible = get_cached_visibility(addr, hpos, n.model)
                                        local col = visible and S.chams_col or S.chams_col2
                                        pcall(function() draw.Chams(hulls, col, col, S.chams_style) end)
                                    else
                                        pcall(function() draw.Chams(hulls, S.chams_col, S.chams_col, S.chams_style) end)
                                    end
                                else
                                    pcall(function() draw.Chams(hulls, S.chams_col, S.chams_col, S.chams_style) end)
                                end
                            else
                                pcall(function() draw.Chams(hulls, S.chams_col, S.chams_style) end)
                            end
                        end
                    end
                end
            end
        end
    end
end

local RADAR_POS = { x = 20, y = 100 }
local hitmarker_time = 0

local function draw_radar()
    if not S.radar then return end
    local size = S.radar_size
    local rng = S.radar_range_studs
    local cx = RADAR_POS.x + size * 0.5
    local cy = RADAR_POS.y + size * 0.5
    local radius = size * 0.5
    draw.circle_filled(cx, cy, radius, {0.03,0.03,0.05,0.85}, 48)
    draw.circle(cx, cy, radius, {0.18,0.72,0.84,0.9}, 48, 1.5)
    draw.line(cx - radius, cy, cx + radius, cy, {0.18,0.72,0.84,0.3}, 1)
    draw.line(cx, cy - radius, cx, cy + radius, {0.18,0.72,0.84,0.3}, 1)
    local lp = entity.get_local_player()
    if not lp or not lp.position then return end
    local lpos = lp.position
    local forward = nil
    if S.radar_rotate then
        local cam_pos = camera.get_position()
        if cam_pos then forward = camera.get_look_vector() end
    end
    local function blip(wx, wy, wz, col, sz)
        local dx = wx - lpos.x
        local dz = wz - lpos.z
        if forward then
            local fx, fz = forward.x, forward.z
            local mag = sqrt(fx*fx + fz*fz)
            if mag > 0.001 then fx, fz = fx/mag, fz/mag end
            local rx = dx * (-fz) + dz * fx
            local rz = dx * fx + dz * fz
            dx, dz = rx, rz
        end
        local d = sqrt(dx*dx + dz*dz)
        if d > rng then return end
        local sx = cx + (dx / rng) * radius
        local sy = cy - (dz / rng) * radius
        draw.circle_filled(sx, sy, sz or 3, col, 12)
    end
    draw.circle_filled(cx, cy, 4, {1,1,1,1}, 12)
    local players = cached_players_frame
    for i=1,#players do
        local p = players[i]
        if p and p.is_valid and p.is_alive and not p.is_local and p.position then
            blip(p.position.x, p.position.y, p.position.z, {1,0.3,0.3,1}, 3)
        end
    end
    for i=1,#npcs_cached do
        local n = npcs_cached[i]
        if n.head and n.head.position then
            local pos = n.head.position
            blip(pos.x, pos.y, pos.z, {1,0.75,0.2,1}, 2.5)
        end
    end
    for i=1,#loot_items do
        local it = loot_items[i]
        if it.part and it.part.position then
            local pos = it.part.position
            blip(pos.x, pos.y, pos.z, {0.35,0.9,0.45,0.8}, 1.5)
        end
    end
end

local function draw_hitmarker()
    if not S.hitmarker then return end
    if not S.aim or not input.is_key_down(S.aim_key) then return end
    local scx, scy = input.get_screen_center()
    if aim_target_ent then hitmarker_time = utility.get_time() end
    if utility.get_time() - hitmarker_time < 0.15 then
        local col = {1,1,1,0.9}
        local off = 8
        draw.line(scx-off, scy-off, scx-off*0.5, scy-off*0.5, col, 1.5)
        draw.line(scx+off, scy-off, scx+off*0.5, scy-off*0.5, col, 1.5)
        draw.line(scx-off, scy+off, scx-off*0.5, scy+off*0.5, col, 1.5)
        draw.line(scx+off, scy+off, scx+off*0.5, scy+off*0.5, col, 1.5)
    end
end

local function apply_misc()
    if S.nograss then
        if original_grass_length == nil then
            local ok, v = pcall(function() return terrain.GetGrassLength() end)
            if ok and v then original_grass_length = v end
        end
        pcall(function() terrain.SetGrassLength(-1) end)
        pcall(function() terrain.SetGrassLength(0) end)
    else
        if original_grass_length ~= nil then
            pcall(function() terrain.SetGrassLength(original_grass_length) end)
        end
    end
end

local SAVE_ITEMS = {
    {"v4_player_enabled","bool"},{"v4_player_box","bool",true},{"v4_player_health","bool"},
    {"v4_player_name","bool",true},{"v4_player_dist","bool",true},{"v4_player_skeleton","bool"},
    {"v4_player_weapon","bool",true},{"v4_player_team_check","bool"},{"v4_player_range","int"},
    {"v4_npc_enabled","bool"},{"v4_npc_box","bool",true},{"v4_npc_health","bool"},
    {"v4_npc_name","bool",true},{"v4_npc_dist","bool",true},{"v4_npc_skeleton","bool"},{"v4_npc_range","int"},
    {"v4_car_enabled","bool"},{"v4_car_col","bool",true},{"v4_car_range","int"},
    {"v4_exit_enabled","bool"},{"v4_exit_col","bool",true},{"v4_exit_range","int"},
    {"v4_corpse_enabled","bool"},{"v4_corpse_name","bool",true},{"v4_corpse_dist","bool",true},
    {"v4_corpse_marker","bool",true},{"v4_corpse_range","int"},
    {"v4_container_enabled","bool"},{"v4_container_col","bool",true},{"v4_container_range","int"},
    {"v4_container_contents","bool"},
    {"v4_quest_enabled","bool"},{"v4_quest_col","bool",true},{"v4_quest_range","int"},
    {"v4_claymore_enabled","bool"},{"v4_claymore_col","bool",true},{"v4_claymore_range","int"},
    {"v4_claymore_names","bool"},
    {"v4_loot_enabled","bool"},{"v4_loot_weapons","bool"},{"v4_loot_armor","bool"},
    {"v4_loot_valuables","bool"},{"v4_loot_meds","bool"},{"v4_loot_ammo","bool"},
    {"v4_loot_other","bool"},{"v4_loot_prefix","bool"},{"v4_loot_range","int"},
    {"v4_chams_enabled","bool"},{"v4_chams_style","int"},
    {"v4_chams_players","bool"},{"v4_chams_npcs","bool"},
    {"v4_chams_gradient","bool"},{"v4_chams_vis_fov","int"},
    {"v4_boss_enabled","bool"},
    {"v4_lt_enabled","bool"},{"v4_lt_max","int"},{"v4_lt_far","int"},
    {"v4_aim_enabled","bool"},{"v4_aim_keybind","key"},
    {"v4_aim_target","int"},{"v4_aim_bone","int"},
    {"v4_aim_fov","int"},{"v4_aim_smooth","int"},
    {"v4_aim_predict","bool"},{"v4_aim_predict_scale","int"},{"v4_aim_lead","bool"},
    {"v4_aim_visible","bool"},{"v4_aim_draw_fov","bool"},{"v4_aim_lock","bool"},
    {"v4_inv_enabled","bool"},{"v4_inv_keybind","key"},{"v4_inv_mode","int"},
    {"v4_inv_guns","bool"},{"v4_inv_max_guns","int"},
    {"v4_inv_armor","bool"},{"v4_inv_max_armor","int"},
    {"v4_inv_valuables","bool"},{"v4_inv_max_valuables","int"},
    {"v4_inv_other","bool"},{"v4_inv_max_other","int"},
    {"v4_inv_equipped","bool"},{"v4_inv_icons","bool"},{"v4_inv_icon_size","int"},
    {"v4_hud_enabled","bool"},{"v4_hud_name","bool"},{"v4_hud_weapon","bool"},
    {"v4_hud_hp","bool"},{"v4_hud_helmet","bool"},{"v4_hud_armor","bool"},
    {"v4_hud_mask","bool"},{"v4_hud_gloves","bool"},{"v4_hud_backpack","bool"},
    {"v4_hud_dist","bool"},{"v4_hud_visible","bool"},{"v4_hud_icons","bool"},{"v4_hud_icon_size","int"},
    {"v4_hud_range","int"},{"v4_hud_offset_y","int"},
    {"v4_radar_enabled","bool"},{"v4_radar_size","int"},{"v4_radar_range","int"},{"v4_radar_rotate","bool"},
    {"v4_hitmarker","bool"},{"v4_nograss","bool"},
    {"v4_ui_opacity","int"},{"v4_ui_border_glow","int"},{"v4_ui_w","int"},{"v4_ui_h","int"},
    {"v4_ui_toggle_key","key"},
}

local function save_config()
    pcall(function()
        local path = get_config_path()
        local dir = path:match("^(.+)\\[^\\]+$")
        if dir then pcall(function() os.execute('mkdir "' .. dir .. '" 2>nul') end) end
        local f = io.open(path, "w")
        if not f then return end
        for _, item in ipairs(SAVE_ITEMS) do
            local id = item[1]
            if item[2] == "key" then
                f:write(id .. "_key=" .. tostring(m_key(id)) .. "\n")
            else
                local val = m_get(id)
                if item[2] == "bool" then f:write(id .. "=" .. (val and "1" or "0") .. "\n")
                else f:write(id .. "=" .. tostring(tonumber(val) or 0) .. "\n") end
                if item[3] then
                    local c = m_col(id)
                    if c then f:write(id .. "_color=" .. string.format("%.4f,%.4f,%.4f,%.4f", c[1] or 1, c[2] or 1, c[3] or 1, c[4] or 1) .. "\n") end
                end
                if item[4] then f:write(id .. "_key=" .. tostring(m_key(id)) .. "\n") end
            end
        end
        local uia = menu.get_color("v4_ui_accent")
        if uia then
            f:write("v4_ui_accent_color=" .. string.format("%.4f,%.4f,%.4f,%.4f", uia[1], uia[2], uia[3], uia[4] or 1) .. "\n")
        end
        f:close()
        print("[Delta V2] Config saved")
    end)
end

local function load_config()
    pcall(function()
        local f = io.open(get_config_path(), "r")
        if not f then return end
        local data = {}
        for line in f:lines() do
            local k, v = line:match("^([^=]+)=(.*)$")
            if k then data[k] = v end
        end
        f:close()
        for _, item in ipairs(SAVE_ITEMS) do
            local id = item[1]
            if item[2] == "key" then
                if data[id .. "_key"] then
                    pcall(function() menu.set_key(id, tonumber(data[id.."_key"]) or 0) end)
                end
            else
                if data[id] then
                    if item[2] == "bool" then pcall(function() menu.set(id, data[id] == "1") end)
                    else pcall(function() menu.set(id, tonumber(data[id]) or 0) end) end
                end
                if item[3] and data[id .. "_color"] then
                    local r,g,b,a = data[id.."_color"]:match("([%d%.%-]+),([%d%.%-]+),([%d%.%-]+),([%d%.%-]+)")
                    if r then pcall(function() menu.set_color(id, {tonumber(r),tonumber(g),tonumber(b),tonumber(a)}) end) end
                end
                if item[4] and data[id.."_key"] then
                    pcall(function() menu.set_key(id, tonumber(data[id.."_key"]) or 0) end)
                end
            end
        end
        if data["v4_ui_accent_color"] then
            local r,g,b,a = data["v4_ui_accent_color"]:match("([%d%.%-]+),([%d%.%-]+),([%d%.%-]+),([%d%.%-]+)")
            if r then
                pcall(function() menu.set_color("v4_ui_accent", {tonumber(r),tonumber(g),tonumber(b),tonumber(a) or 1}) end)
            end
        end
        print("[Delta V2] Config loaded")
    end)
end

menu.add_button("Config", "Config", "v4_save_btn", "Save Config", save_config)
menu.add_button("Config", "Config", "v4_load_btn", "Load Config", load_config)

thread.create(update_camera, 33)
thread.create(function() refresh_folders() scan_npcs() end, 1500)
thread.create(scan_loot, 1500)
thread.create(scan_corpses, 3000)
thread.create(scan_exits, 5000)
thread.create(function()
    if not S.car then return end
    scan_cars()
end, 5000)
thread.create(scan_bosses, 2500)
thread.create(scan_players_inv, 2500)
thread.create(lt_scan, 2000)
thread.create(scan_player_inventories, 2000)
thread.create(scan_containers, 4000)
thread.create(scan_quests, 4000)
thread.create(scan_claymores, 4000)
thread.create(refresh_settings, 500)

_G.on_frame = function()
    cached_players_frame = entity.get_players() or {}
    update_camera()

    tick_and_render()

    local ui_blocking = false
    if base_ui.state.open then
        local mx, my = mouse_pos()
        if in_rect(mx, my, base_ui.state.x, base_ui.state.y, base_ui.state.w, base_ui.state.h) then
            ui_blocking = true
        end
    end

    if S.player then draw_player_esp() end
    if S.npc then draw_npc_esp() end
    if S.car then draw_car_esp() end
    if S.exit_enabled then draw_exit_esp() end
    if S.corpse then draw_corpse_esp() end
    if S.loot then draw_loot_esp() end
    if S.container then draw_container_esp() end
    if S.quest then draw_quest_esp() end
    if S.claymore then draw_claymore_esp() end
    if S.chams then draw_chams() end
    if S.aim then run_aimbot() end
    if S.radar then draw_radar() end
    if S.hitmarker then draw_hitmarker() end
    if S.nograss then apply_misc() end

    if not ui_blocking then
        if S.boss then draw_boss_tracker() end
        if S.lt then draw_loot_tracker() end
        if S.hud then draw_target_hud() end
        if S.inv then draw_inventory_panel() end
    end
end
_G.OnFrame = _G.on_frame
_G.onFrame = _G.on_frame

refresh_folders()
scan_npcs()
scan_loot()
scan_corpses()
scan_exits()
scan_cars()
scan_bosses()
scan_players_inv()
scan_player_inventories()
scan_containers()
scan_quests()
scan_claymores()
update_camera()
refresh_settings()
load_config()

local boot_toggle_key = m_key("v4_ui_toggle_key")
if boot_toggle_key and boot_toggle_key ~= 0 then
    base_ui.state.toggle_key = boot_toggle_key
end

if base_ui.state.open then
    set_cursor_visible(true)
end

end
