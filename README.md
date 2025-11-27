# Pyodide Bioinformatics Demo

This app runs Python entirely in your browser using Pyodide. It cleans a DNA sequence, computes GC content and related summary stats, and prints the reverse complement in a FASTA-style format.

---

## How does the app work?

Look at the `<script>` block at the bottom of the HTML file. The flow is:

1. **Load Pyodide and store a promise**

   ```javascript
   let pyodideReadyPromise = loadPyodide();

   const statusEl = document.getElementById("status");
   const outputEl = document.getElementById("output");
   const analyzeButton = document.getElementById("analyzeButton");
   ```

   `loadPyodide()` starts downloading and initializing the Python runtime compiled to WebAssembly. It returns a promise that resolves to the `pyodide` object once everything is ready.

2. **Set up Pyodide and define Python helpers**

   ```javascript
   async function setupPyodide() {
     try {
       const pyodide = await pyodideReadyPromise;
       const packages = []; // Optionally add packages
       
       if (packages.length > 0) {
         await pyodide.loadPackage("micropip");
         const micropip = pyodide.pyimport("micropip");
         await Promise.all(
           packages.map(pkg => micropip.install(pkg))
         );
       }

       // Define analysis code once in the Python environment
       pyodide.runPython(`

        from textwrap import wrap
        
        def gc_content(seq: str):
           ...
        def reverse_complement(seq: str) -> str:
           ...

        def analyze_sequence(raw_seq: str) -> str:
           ...
       `);

       statusEl.textContent = "Pyodide ready.";
       analyzeButton.disabled = false;
     } catch (err) {
       console.error(err);
       statusEl.textContent = "Error loading Pyodide. See console for details.";
     }
   }

   setupPyodide();
   ```  

Here we:

- Wait for the Pyodide runtime.
- Optionally install extra Python packages with `micropip`.
- Define `gc_content`, `reverse_complement`, and `analyze_sequence` in the Python environment once at startup.
- Enable the button when everything is ready.

3. **Wire up the button to run the analysis**

```javascript
async function analyze() {
  analyzeButton.disabled = true;
  statusEl.textContent = "Running analysis…";

  try {
    const pyodide = await pyodideReadyPromise;
    const seq = document.getElementById("sequence").value;

    // Pass the JS string into the Python environment safely
    pyodide.globals.set("raw_seq_js", seq);

    const result = pyodide.runPython(`
# 'raw_seq_js' is injected from JavaScript
analyze_sequence(raw_seq_js)
    `);

    outputEl.textContent = result;
    statusEl.textContent = "Done.";
  } catch (err) {
    console.error(err);
    outputEl.textContent = "Error during analysis:\n" + err;
    statusEl.textContent = "Error.";
  } finally {
    analyzeButton.disabled = false;
  }
}

analyzeButton.addEventListener("click", analyze);
````

When the user clicks "Analyze sequence", this function:

* Reads the sequence from the `<textarea>`.
* Injects it into the Python global namespace.
* Calls `analyze_sequence` from Python.
* Displays the returned text in the `<pre>` results box.

---

## How do data enter WASM?

The DNA sequence is typed or pasted into the `<textarea id="sequence">` element. On click:

```javascript
const seq = document.getElementById("sequence").value;

// Pass the JS string into the Python environment safely
pyodide.globals.set("raw_seq_js", seq);
```

Key points:

* `seq` is a plain JavaScript string.
* `pyodide.globals.set("raw_seq_js", seq)` copies that string into the Python environment as a global variable named `raw_seq_js`.
* In the subsequent `runPython` call, Python code can use `raw_seq_js` directly:

  ```python
  # inside the runPython string
  analyze_sequence(raw_seq_js)
  ```

Inside `analyze_sequence` the data are cleaned and normalized:

* FASTA headers starting with `>` are removed.
* Spaces and newlines are stripped.
* Characters are uppercased and non A/C/G/T bases are discarded.

---

## How do data exit WASM?

The Python function `analyze_sequence` returns a single string that already contains all formatted output:

```python
def analyze_sequence(raw_seq: str) -> str:
    ...
    return "\n".join(lines_out)
```

On the JavaScript side:

```javascript
const result = pyodide.runPython(`
# 'raw_seq_js' is injected from JavaScript
analyze_sequence(raw_seq_js)
`);

outputEl.textContent = result;
statusEl.textContent = "Done.";
analyzeButton.disabled = false;
```

Here:

* `pyodide.runPython(...)` executes the Python code synchronously and returns the Python result converted to a JavaScript type.
* Since the Python function returns a plain string, `result` is a JavaScript string.
* That string is assigned directly to `outputEl.textContent`, so it appears in the `<pre>` output block.

No manual marshaling of complex objects is needed in this example, because we only pass strings in and out.

---

## Questions

### What is Pyodide?

Pyodide is a distribution of Python compiled to WebAssembly that runs in the browser. It provides:

* A Python interpreter with many standard scientific packages.
* A bridge between JavaScript and Python objects.
* The ability to install extra pure Python packages via `micropip`.

All computation in this app happens client side. The sequence never leaves the browser.

---

### What does `pyodideReadyPromise` represent?

```javascript
let pyodideReadyPromise = loadPyodide();
```

* `loadPyodide()` starts loading and initializing the interpreter.
* It returns a promise that resolves to the `pyodide` object when ready.
* Using `await pyodideReadyPromise` ensures that any use of `pyodide` happens only after initialization has completed.

This avoids race conditions where you try to run Python before Pyodide is loaded.

---

### What does `pyodide.runPython()` do?

`pyodide.runPython(code)`:

* Executes the given Python code string inside the Pyodide interpreter.
* Can access any variables defined previously or injected from JavaScript.
* Returns the final expression value from the executed code, converted to a JavaScript value when possible (for example strings, numbers, lists of simple types).

In this app it is used in two ways:

1. At startup, to define helper functions:

   ```javascript
   pyodide.runPython(`
   ```

from textwrap import wrap
...
def analyze_sequence(raw_seq: str) -> str:
...
`);

````

Here we do not use the return value, we just define functions in the Python environment.

2. On button click, to call `analyze_sequence` and get its return value:

```javascript
const result = pyodide.runPython(`
analyze_sequence(raw_seq_js)
`);
````

---

### What does `pyodide.globals.set(...)` do?

```javascript
pyodide.globals.set("raw_seq_js", seq);
```

* `pyodide.globals` is like Python’s global namespace.
* `.set(name, value)` assigns a JavaScript value to a Python variable in that namespace.
* After this call, Python code can use `raw_seq_js` as a normal `str`.

This is the main bridge used here to pass user input into WASM.

---

### What does `analyzeButton.disabled = false` mean?

The `disabled` property on a `<button>` element controls whether the button is clickable:

* `analyzeButton.disabled = true` disables the button so it cannot be clicked.
* `analyzeButton.disabled = false` re-enables it.

In this app:

* The button starts out disabled until Pyodide is fully loaded.
* It is temporarily disabled while analysis is running to prevent multiple overlapping runs.
* It is re-enabled in the `finally` block so the user can run another analysis.
