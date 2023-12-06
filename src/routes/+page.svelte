<script>
    import {onMount} from 'svelte';
    
    // search values
    let prevQuery;
    let searchQuery;
    // let searchQuery;
    let replaceText;
    let numMatches;
    let foundLines = [];

    // editor objects
    let Monaco;
    let editor;
    let divEl;
    var welcomeText =
        "<catalog>"
        + "<book id=\"bk01\">"
        + "<author>Gambardella, Matthew</author>"
        + "<title>Developer's Guide</title>"
        + "<genre>Computer</genre>"
        + "<publish_date>2000-10-01</publish_date>"
        + "<description>An in-depth look at creating applications with XML.</description>"
        + "</book>"
        + "<book id=\"bk02\">"
        + "<author>Ralls, Kim</author>"
        + "<title>Midnight Rain</title>"
        + "<genre>Fantasy</genre>"
        + "<publish_date>2000-12-16</publish_date>"
        + "<description>A former architect battles corporate zombies, an evil sorceress, and her own childhood to become queen of the world.</description>"
        + "</book>"
        + "</catalog>";
    onMount(async () => {
        Monaco = await import("monaco-editor");
        editor = Monaco.editor.create(divEl, {
            language: 'xml',
            automaticLayout: true,
            autoIndent: 'full',
            formatOnPaste: true,
            value: formatXml(welcomeText),
            theme: "vs-dark", 
            wordWrap: "on"
        });
    });

// upload() xml to editor
function uploadXml() {
    editor.getModel().setValue("");
    let editorContent;
    let file = document.getElementById('file-upload').files[0];
    let fr = new FileReader();
    fr.onload = e => {
        editorContent = e.target.result;
        editor.trigger('keyboard', 'type', { text: editorContent });
    }
    fr.readAsText(file, 'UTF-8');

    
}

// find() a property in the file
function findProperty() {
    console.log("previous query " + prevQuery)
    console.log("search query " + searchQuery)
    var matches;

    // clear previous match inline classes if exists
    if (prevQuery !== undefined) {
        var decorationIds = [];
        matches = editor.getModel().findMatches(prevQuery);
        matches.forEach(match => {
            var decorations = editor.getDecorationsInRange(match.range);
            decorations.forEach(decoration => {
                decorationIds.push(decoration.id);
            })
        })
        editor.removeDecorations(decorationIds);
    }

    // set inline classname of found query 
    matches = editor.getModel().findMatches(searchQuery);
    if (matches.length == 0) {
        return 0;
    } else {
        foundLines = [];
        matches.forEach(match => {
            foundLines.push(match.range.startLineNumber)
            editor.createDecorationsCollection(
                [
                    {
                        range: match.range,
                        options:
                        {
                            // toggle to highlight whole line vs match
                            isWholeLine: true,
                            inlineClassName: "searchQuerySelection"
                        }
                    }
                ]);
        });
        prevQuery = searchQuery;
        
        numMatches = matches.length;
    }
}

// find/replace within the editor
function findReplace() {
    if(replaceText != "") {
        let matches = editor.getModel().findMatches(searchQuery);
        let startLineNo;
        let startCol;
        let endLineNo;
        let endCol;
        if (matches.length > 0) {
            // console.log('num matches ' + matches.length);
            //matches.forEach(match => {
            //    editor.executeEdits(null, [
            //        {
            //            range: match.range,
            //            text: replaceText
            //        }
            //    ]);
            //})
            for (var i = 0; i < matches.length; i++) {
                startLineNo = matches[i].range.startLineNumber;
                startCol = matches[i].range.startColumn;
                endLineNo = matches[i].range.endLineNumber;
                endCol = matches[i].range.endColumn;
                
                editor.executeEdits(null, [
                    {
                        range: new Monaco.Range(startLineNo, startCol, endLineNo, endCol),
                        text: replaceText
                    }
                ]);
                
            } 
        }
        prevQuery = replaceText;
    }
    
}

// toggle word wrap
function toggleWordWrap(e) {


    if(e.target.value === "on") {
        editor.updateOptions({wordWrap: "off"})
        e.target.value = "off";
    } else {
        editor.updateOptions({wordWrap: "on"})
        e.target.value = "on";
    }
}
// toggle dark mode
function toggleDarkMode(e) {
    if(e.target.value === "on") {
        Monaco.editor.setTheme("vs-light");
        e.target.value = "off";
    } else {
        Monaco.editor.setTheme("vs-dark");
        e.target.value = "on";
    }
}

/******************
 * HELPER FUNCTIONS
 ******************/
// formatting xml
// thanks to Japheth Obala's answer at https://stackoverflow.com/questions/57039218/doesnt-monaco-editor-support-xml-language-by-default
function formatXml(xml) {
    const PADDING = ' '.repeat(2);
    const reg = /(>)(<)(\/*)/g;
    let pad = 0;

    xml = xml.replace(reg, '$1\r\n$2$3');

    return xml.split('\r\n').map((node, index) => {
        let indent = 0;
        if (node.match(/.+<\/\w[^>]*>$/)) {
            indent = 0;
        } else if (node.match(/^<\/\w/) && pad > 0) {
            pad -= 1;
        } else if (node.match(/^<\w[^>]*[^/]>.*$/)) {
            indent = 1;
        } else {
            indent = 0;
        }

        pad += indent;

        return PADDING.repeat(pad - indent) + node;
    }).join('\r\n');
}
</script>

<main>
    <header>
        <h1>&lt;xml-edit /&gt;</h1>
    </header>

    <div id='editor' bind:this={divEl} />

    <div id='action-container'>
        <div id='options-container'>
            <label>
                <input type='checkbox' checked={true} on:change={(e)=> toggleDarkMode(e)}/>
                dark mode
            </label>
            <label>
                <input type='checkbox' checked={true} on:change={(e)=> toggleWordWrap(e)}/>
                word wrap
            </label>
        </div>
        <label class='upload-btn btn'>
            <input on:change={() => uploadXml()} id='file-upload' type='file' style='display: none;' accept='.xml' />
            Upload
            <sub>(Accepts .xml format...)</sub>
        </label>
        <div id='find-container'>
            <div id='find-form'>
                <form  on:submit={() => findProperty()}>
                    <input bind:value={searchQuery} class='find-field field' type='text'  />
                    <button class='find-btn btn' type='submit'>Find</button>
                    <p>{numMatches != undefined ? numMatches + ' match(es)' : '' }</p>
                </form>
                <form on:submit={() => findReplace()}>
                    <input bind:value={replaceText} class='field' name='input-replace' type='text' />
                    <button class='btn' type='submit'>Replace</button>
                </form>
            </div>
        </div>

    </div>
</main>

<style >
    main {
        min-height: 100vh;
        box-shadow: 2px 2px 10px rgba(0 0 0 / .15), 
        -2px -2px 10px rgba(0 0 0 / .15);
    }
    header {
        padding: .5rem 1rem;
        display: flex;
        justify-content: space-between;
        align-items: center;
        background: navy;
        color: white;
    }
    #editor {
        width: 100%;
        height: 80%;
        min-height: 600px;
    }
    .btn {
        min-width: 120px;
        box-shadow: 2px 2px 2px rgba(0 0 0 / .15);
    }
    .btn:hover {
        cursor: pointer;
    }
    .btn, .field {
        padding: .5rem;
        background: white;
    }
    .upload-btn {
        position: relative;
        max-width: 120px;
        margin-bottom: .5rem;
        display: flex;
        justify-content: center;
        background: white;
        color: black;
        font-size: 12px;
    }
    .upload-btn > sub {
        position: absolute;
        right: -7rem;
        bottom: 0;
    }
    #options-container {
        margin-bottom: .5rem;
    }
    #action-container {
        padding: 1rem;
    }
    #find-container {
        width: fit-content;
    }
    #find-form {
        display: flex;
        flex-direction: column;
        gap: .5rem;
    }
    form {
        position: relative;
    }
    form > p {
        position: absolute;
        right: -6rem;
        top: .5rem;
    }

</style>

