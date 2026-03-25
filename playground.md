---
layout: default
title: Monkey Playground
permalink: /playground/
---

<style>
.playground-container { max-width: 900px; margin: 0 auto; padding: 20px; font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, sans-serif; }
.editor-container { display: flex; gap: 16px; margin-bottom: 16px; }
.editor-pane, .output-pane { flex: 1; min-height: 300px; }
.editor-pane textarea, .output-pane pre { width: 100%; height: 300px; font-family: 'SF Mono', monospace; font-size: 14px; padding: 12px; border: 1px solid #ddd; border-radius: 8px; resize: vertical; box-sizing: border-box; }
.editor-pane textarea { background: #1e1e2e; color: #cdd6f4; border-color: #45475a; }
.output-pane pre { background: #f5f5f5; overflow: auto; margin: 0; }
.controls { display: flex; gap: 8px; margin-bottom: 12px; align-items: center; flex-wrap: wrap; }
.controls button { padding: 8px 16px; border: none; border-radius: 6px; cursor: pointer; font-size: 14px; font-weight: 500; }
.run-btn { background: #a6e3a1; color: #1e1e2e; }
.bench-btn { background: #89b4fa; color: #1e1e2e; }
.clear-btn { background: #f38ba8; color: #1e1e2e; }
.stats { color: #666; font-size: 13px; margin-left: auto; }
label { font-size: 13px; display: flex; align-items: center; gap: 4px; }
select { font-size: 13px; padding: 4px 8px; border-radius: 4px; border: 1px solid #ddd; }
h1 { margin-bottom: 4px; }
.subtitle { color: #666; margin-bottom: 20px; }
@media (max-width: 768px) { .editor-container { flex-direction: column; } }
</style>

<div class="playground-container">
<h1>🐵 Monkey Playground</h1>
<p class="subtitle">A tracing JIT compiler in JavaScript — <a href="https://github.com/henry-the-frog/monkey-lang">source</a></p>

<div class="controls">
<button class="run-btn" onclick="runCode()">▶ Run</button>
<button class="bench-btn" onclick="benchCode()">⏱ Benchmark</button>
<button class="clear-btn" onclick="clearOutput()">✕ Clear</button>
<label>Engine: <select id="engine"><option value="jit" selected>JIT</option><option value="vm">VM</option></select></label>
<label><input type="checkbox" id="stdlib" checked> stdlib</label>
<label>Example: <select id="examples" onchange="loadExample()"><option value="">— select —</option><option value="fibonacci">Fibonacci</option><option value="fizzbuzz">FizzBuzz</option><option value="arrays">Arrays</option><option value="closures">Closures</option><option value="forloops">For Loops</option><option value="templates">Templates</option><option value="mandelbrot">Mandelbrot</option><option value="sorting">Sorting</option></select></label>
<span class="stats" id="stats"></span>
</div>

<div class="editor-container">
<div class="editor-pane">
<textarea id="code" spellcheck="false">// Fibonacci — try changing the number!
let fib = fn(n) {
  if (n < 2) { return n; }
  fib(n - 1) + fib(n - 2)
};

puts("fib(25) = " + str(fib(25)))</textarea>
</div>
<div class="output-pane">
<pre id="output">Press ▶ Run or Ctrl+Enter to execute</pre>
</div>
</div>
</div>

<script src="/assets/js/monkey-bundle.js"></script>
<script>
const EXAMPLES={fibonacci:`let fib = fn(n) {\n  if (n < 2) { return n; }\n  fib(n - 1) + fib(n - 2)\n};\nputs("fib(25) = " + str(fib(25)))\nputs("fib(30) = " + str(fib(30)))`,fizzbuzz:`let i = 1;\nwhile (i < 31) {\n  if (i % 15 == 0) { puts("FizzBuzz"); }\n  if (i % 15 != 0) {\n    if (i % 3 == 0) { puts("Fizz"); }\n    if (i % 3 != 0) {\n      if (i % 5 == 0) { puts("Buzz"); }\n      if (i % 5 != 0) { puts(str(i)); }\n    }\n  }\n  i = i + 1;\n}`,arrays:`let nums = range(100);\nlet sum = reduce(nums, 0, fn(a, x) { a + x });\nputs("Sum 0..99: " + str(sum));\nlet big = filter(nums, fn(x) { x > 90 });\nputs("Numbers > 90: " + str(len(big)));\nputs("5! = " + str(reduce([1,2,3,4,5], 1, fn(a,x) { a*x })))`,forloops:`// For loops, for-in, break, continue\nlet sum = 0;\nfor (let i = 0; i < 10; i += 1) {\n  sum += i;\n}\nputs(`Sum 0..9: ${sum}`);\n\n// For-in iteration\nlet fruits = ["apple", "banana", "cherry"];\nfor (f in fruits) {\n  puts(`I like ${f}`);\n}\n\n// Break and continue\nlet s = 0;\nfor (let i = 0; i < 100; i += 1) {\n  if (i % 2 == 0) { continue; }\n  if (i > 20) { break; }\n  s += i;\n}\nputs(`Sum of odd numbers 1..19: ${s}`)`,templates:`// String interpolation with backtick templates\nlet name = "Monkey";\nlet version = 2;\nputs(`Welcome to ${name} v${version}!`);\n\n// Expressions in templates\nlet a = 7;\nlet b = 8;\nputs(`${a} * ${b} = ${a * b}`);\n\n// String multiplication\nputs("ha" * 5);\nputs("-" * 40);\n\n// Negative indexing\nlet arr = [10, 20, 30, 40, 50];\nputs(`Last: ${arr[-1]}, Second-to-last: ${arr[-2]}`);\nlet word = "hello";\nputs(`Last char: ${word[-1]}`)`,mandelbrot:`// ASCII Mandelbrot — fixed-point arithmetic\nlet SCALE = 1000;\nlet max_iter = 30;\nlet width = 60;\nlet height = 25;\nlet chars = " .-+*#@";\n\nfor (let row = 0; row < height; row += 1) {\n  let line = "";\n  for (let col = 0; col < width; col += 1) {\n    let x0 = -2000 + col * 3000 / width;\n    let y0 = -1000 + row * 2000 / height;\n    let x = 0;\n    let y = 0;\n    let iter = 0;\n    while (iter < max_iter) {\n      let x2 = x * x / SCALE;\n      let y2 = y * y / SCALE;\n      if (x2 + y2 > 4 * SCALE) { break; }\n      let xtemp = x2 - y2 + x0;\n      y = 2 * x * y / SCALE + y0;\n      x = xtemp;\n      iter += 1;\n    }\n    if (iter == max_iter) {\n      line = line + "#";\n    } else {\n      let ci = iter * len(chars) / max_iter;\n      if (ci >= len(chars)) { ci = len(chars) - 1; }\n      line = line + chars[ci];\n    }\n  }\n  puts(line);\n}`,sorting:`// Insertion sort (immutable arrays)\nlet insertion_sort = fn(arr) {\n  let sorted = [];\n  for (x in arr) {\n    let inserted = false;\n    let result = [];\n    for (s in sorted) {\n      if (!inserted) {\n        if (x < s) {\n          result = push(result, x);\n          inserted = true;\n        }\n      }\n      result = push(result, s);\n    }\n    if (!inserted) {\n      result = push(result, x);\n    }\n    sorted = result;\n  }\n  sorted\n};\n\nlet data = [38, 27, 43, 3, 9, 82, 10];\nputs("Input:  " + str(data));\nputs("Sorted: " + str(insertion_sort(data)));\n\nlet words = ["banana", "apple", "cherry", "date"];\nputs("\\nWords:  " + str(words));\nputs("Sorted: " + str(insertion_sort(words)))`,closures:`let makeAdder = fn(x) { fn(y) { x + y } };\nlet add5 = makeAdder(5);\nputs("add5(3) = " + str(add5(3)));\nputs("add5(10) = " + str(add5(10)))`};
const STDLIB=Monkey.STDLIB_SOURCE;

function transpileCode() {
  const code = document.getElementById('code').value;
  const outputEl = document.getElementById('output');
  try {
    const lexer = new Monkey.Lexer(code);
    const parser = new Monkey.Parser(lexer);
    const program = parser.parseProgram();
    if (parser.errors.length > 0) {
      outputEl.textContent = 'Parse errors:\n' + parser.errors.join('\n');
      return;
    }
    const transpiler = new Monkey.Transpiler();
    const js = transpiler.transpile(program);
    outputEl.textContent = '// Transpiled JavaScript:\n\n' + js;
  } catch(e) {
    outputEl.textContent = 'Error: ' + e.message;
  }
}

function runCode(){const o=document.getElementById('output');o.textContent='';const s=document.getElementById('stdlib').checked;const e=document.getElementById('engine').value;let c=document.getElementById('code').value;if(s)c=STDLIB+'\n'+c;const lines=[];const ol=console.log;console.log=(...a)=>lines.push(a.map(x=>String(x)).join(' '));try{const l=new Monkey.Lexer(c);const p=new Monkey.Parser(l);const pr=p.parseProgram();if(p.errors.length>0){o.textContent='Parse errors:\n'+p.errors.join('\n');console.log=ol;return}const co=new Monkey.Compiler();co.compile(pr);const vm=new Monkey.VM(co.bytecode());if(e==='jit')vm.enableJIT();const t=performance.now();vm.run();const el=performance.now()-t;const r=vm.lastPoppedStackElem();if(r&&r!==Monkey.NULL)lines.push('→ '+r.inspect());const tc=e==='jit'&&vm.jit?vm.jit.traces.size:0;document.getElementById('stats').textContent=el.toFixed(2)+'ms'+(tc>0?' | '+tc+' trace'+(tc>1?'s':''):'');o.textContent=lines.join('\n')||'(no output)'}catch(e){o.textContent='Error: '+e.message}console.log=ol}
function benchCode(){const o=document.getElementById('output');o.textContent='Benchmarking...';const s=document.getElementById('stdlib').checked;let c=document.getElementById('code').value;if(s)c=STDLIB+'\n'+c;const ol=console.log;console.log=()=>{};const N=50;let vt=0,jt=0;try{for(let i=0;i<N;i++){const l=new Monkey.Lexer(c);const p=new Monkey.Parser(l);const pr=p.parseProgram();const co=new Monkey.Compiler();co.compile(pr);const vm=new Monkey.VM(co.bytecode());const s=performance.now();vm.run();vt+=performance.now()-s}for(let i=0;i<N;i++){const l=new Monkey.Lexer(c);const p=new Monkey.Parser(l);const pr=p.parseProgram();const co=new Monkey.Compiler();co.compile(pr);const vm=new Monkey.VM(co.bytecode());vm.enableJIT();const s=performance.now();vm.run();jt+=performance.now()-s}const va=vt/N,ja=jt/N,sp=va/ja;o.textContent='Benchmark ('+N+' iterations)\n\n  VM:      '+va.toFixed(3)+'ms avg\n  JIT:     '+ja.toFixed(3)+'ms avg\n  Speedup: '+sp.toFixed(2)+'x'}catch(e){o.textContent='Error: '+e.message}console.log=ol}
function clearOutput(){document.getElementById('output').textContent='';document.getElementById('stats').textContent=''}
function loadExample(){const s=document.getElementById('examples');if(EXAMPLES[s.value])document.getElementById('code').value=EXAMPLES[s.value];s.value=''}
document.getElementById('code').addEventListener('keydown',e=>{if((e.ctrlKey||e.metaKey)&&e.key==='Enter'){e.preventDefault();runCode()}if(e.key==='Tab'){e.preventDefault();const t=e.target,s=t.selectionStart;t.value=t.value.substring(0,s)+'  '+t.value.substring(t.selectionEnd);t.selectionStart=t.selectionEnd=s+2}});
</script>
