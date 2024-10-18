# How to Replace Textbox with JSON Editor for JSON Fields in PHPMaker

1. **Include JSON Editor Library**
   Add these lines to your project's main template:
   ```html
   <link href="https://cdnjs.cloudflare.com/ajax/libs/jsoneditor/9.10.0/jsoneditor.min.css" rel="stylesheet" type="text/css">
   <script src="https://cdnjs.cloudflare.com/ajax/libs/jsoneditor/9.10.0/jsoneditor.min.js"></script>
   ```

2. **Modify Table-Specific JavaScript**
   In your table-specific JavaScript file (e.g., `record_metadataedit.js`), add:

   ```javascript
   loadjs.ready("load", function() {
     function initJsonEditor() {
       var textarea = document.querySelector('textarea[data-field="x_metadata_schema"]');
       if (!textarea) return;

       var container = document.createElement('div');
       container.id = 'jsoneditor';
       container.style.width = '100%';
       container.style.height = '400px';

       textarea.parentNode.insertBefore(container, textarea);
       textarea.style.display = 'none';

       var editor = new JSONEditor(container, {
         mode: 'tree',
         modes: ['tree', 'view', 'form', 'code', 'text'],
         onChangeText: function(jsonString) {
           textarea.value = jsonString;
         }
       });

       try {
         editor.set(JSON.parse(textarea.value));
       } catch (e) {
         console.error("Invalid JSON:", e);
         editor.set({});
       }
     }

     initJsonEditor();

     $(document).on('refresh.ew', function(e, data) {
       if (data.name === "frecord_metadataedit") {
         initJsonEditor();
       }
     });
   });
   ```

3. **Key Points**
   - No changes to PHPMaker-generated view files needed.
   - Script automatically replaces textarea with JSON editor.
   - Original textarea is hidden but still updates with changes.
   - JSON editor initializes with existing textarea content.
   - Works with dynamic form refreshes in PHPMaker.

4. **Customization**
   - Adjust the textarea selector if field name differs.
   - Modify JSON editor options as needed (e.g., height, modes).

5. **Troubleshooting**
   - Ensure JSON Editor library is correctly loaded.
   - Check browser console for any JavaScript errors.
   - Verify that the field name in the script matches your JSON field.

This approach provides a seamless integration of the JSON editor without modifying PHPMaker's generated HTML, making it easier to maintain across PHPMak
