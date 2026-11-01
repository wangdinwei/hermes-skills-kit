---
name: admin-json-editor-to-visual-list-editor
description: 将管理后台的 JSON textarea 编辑器（方向/影片名/角色配置等）改造为可视化列表/Tab编辑器，非技术管理员可直接增删改，不用写 JSON 格式
---

# Admin JSON Editor → Visual List/Tab Editor

Replace raw `<textarea>` JSON editors in Flask admin pages with visual editors that let non-technical admins edit data directly.

## When to use

An admin page has a modal/panel that shows a `<textarea>` with JSON content like `["item1", "item2"]` or `[{...}, {...}]`. The admin needs to edit this data but shouldn't need to write JSON.

## Architecture

The pattern uses:
1. A single modal with a dynamic content area (`#modalEditorArea`)
2. Per-data-type render functions that build HTML inline
3. An `_editorData` array holding the working copy
4. A gather step before save that collects data from the DOM back into `_editorData`

## Three editor types

### 1. List editor (for string arrays)
- Suitable for: direction names, film titles, tag lists, any flat string array
- Each item: numbered row with `<input>` + delete ✕ button
- Bottom: add row with text input + "＋ 添加" button
- Save: send `_editorData` (array of strings) directly

```javascript
function renderListEditor(data) {
  _editorData = (data && Array.isArray(data)) ? data.slice() : [];
  var html = '';
  _editorData.forEach(function(v, i) {
    html += '<div style="display:flex;align-items:center;gap:8px;margin-bottom:6px">';
    html += '<span>' + (i+1) + '.</span>';
    html += '<input style="flex:1" value="' + escAttr(v) + '" onchange="_editorData['+i+']=this.value"/>';
    html += '<button onclick="_editorData.splice('+i+',1);renderListEditor(_editorData)">✕</button>';
    html += '</div>';
  });
  html += '<div style="display:flex;gap:8px;margin-top:8px">';
  html += '<input id="newItemInput" placeholder="添加..." onkeydown="if(event.key===\'Enter\')addListItem()"/>';
  html += '<button onclick="addListItem()">＋ 添加</button>';
  html += '</div>';
  document.getElementById('modalEditorArea').innerHTML = html;
}
```

### 2. Tab-based role group editor (for structured object arrays)
- Suitable for: character sets, config groups, any array of structured objects
- Tab bar at top: one tab per group + "＋ 新组" button
- Each group rendered as a card with editable labeled sections
- Use `prompt()` for inline name edits (simple, works without third-party)

### 3. Tab-based record editor (fallback for any structured array)
- Tab bar + JSON textarea per record
- Each tab: textarea with the JSON of that record

## Key implementation details

### Global state
```javascript
var _editingTid = null;    // which template/item is being edited
var _editingType = null;   // what data type (directions/films/char_config)
var _editorData = [];      // working copy of the data array
```

### Modal structure
```html
<div id="modalEditorArea" style="min-height:200px;max-height:55vh;overflow-y:auto"></div>
```
- Replace the old textarea with this dynamic container
- Bottom: Cancel + Save buttons only (no "regenerate" since data lives in DB)

### Save flow
```javascript
function saveEditModal() {
  // 1. Sync data (e.g., rebuild all_names from male_leads+female_leads+supporting)
  // 2. POST _editorData to the existing save API
  // 3. On success: close modal + refresh page
}
```

### Data sync pattern (for role groups)
Before saving, rebuild derived fields from visual fields:
```javascript
_editorData.forEach(function(s) {
  s.all_names = [];
  (s.male_leads || []).forEach(function(n){ s.all_names.push(n); });
  (s.female_leads || []).forEach(function(n){ s.all_names.push(n); });
  (s.supporting || []).forEach(function(n){ s.all_names.push(n); });
});
```

## Pitfalls

1. **XSS**: Use `escAttr()` helper for any values interpolated into HTML attributes. Otherwise `"` in a direction name breaks the input value attribute.
2. **Closure in onclick**: When generating HTML with inline `onclick`, use `_editorData[i]` by index (not `v`) because the closure captures by reference and `i` is stable.
3. **Rebind event handlers**: Since content is rebuilt via `innerHTML`, all event handlers must be inline (`onclick`, `onchange`).
4. **Scroll overflow**: Set `max-height:55vh;overflow-y:auto` on the editor area so long lists don't push buttons off-screen.
5. **Save timing**: Use `setTimeout(function(){ closeEditModal(); refreshStatus(); }, 1200)` to let the user see the success message before the modal closes.

## CSS style organization
Store reusable styles in a `_EDIT_STYLE` object:
```javascript
var _EDIT_STYLE = {
  item: 'display:flex;align-items:center;gap:8px;...',
  input: 'flex:1;background:...',
  del: 'background:rgba(239,68,68,0.15)...',
  addRow: 'display:flex;gap:8px;margin-top:8px',
  // etc.
};
```
Then reference as `style="' + _EDIT_STYLE.item + '"` in HTML generation. Keeps the JS readable.
