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
<label>Example: <select id="examples" onchange="loadExample()"><option value="">— select —</option><option value="fibonacci">Fibonacci</option><option value="fizzbuzz">FizzBuzz</option><option value="arrays">Arrays</option><option value="closures">Closures</option></select></label>
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
const EXAMPLES={fibonacci:`let fib = fn(n) {\n  if (n < 2) { return n; }\n  fib(n - 1) + fib(n - 2)\n};\nputs("fib(25) = " + str(fib(25)))\nputs("fib(30) = " + str(fib(30)))`,fizzbuzz:`let i = 1;\nwhile (i < 31) {\n  if (i % 15 == 0) { puts("FizzBuzz"); }\n  if (i % 15 != 0) {\n    if (i % 3 == 0) { puts("Fizz"); }\n    if (i % 3 != 0) {\n      if (i % 5 == 0) { puts("Buzz"); }\n      if (i % 5 != 0) { puts(str(i)); }\n    }\n  }\n  i = i + 1;\n}`,arrays:`let nums = range(100);\nlet sum = reduce(nums, 0, fn(a, x) { a + x });\nputs("Sum 0..99: " + str(sum));\nlet big = filter(nums, fn(x) { x > 90 });\nputs("Numbers > 90: " + str(len(big)));\nputs("5! = " + str(reduce([1,2,3,4,5], 1, fn(a,x) { a*x })))`,closures:`let makeAdder = fn(x) { fn(y) { x + y } };\nlet add5 = makeAdder(5);\nputs("add5(3) = " + str(add5(3)));\nputs("add5(10) = " + str(add5(10)))`};
const STDLIB=Monkey.STDLIB_SOURCE;
function runCode(){const o=document.getElementById('output');o.textContent='';const s=document.getElementById('stdlib').checked;const e=document.getElementById('engine').value;let c=document.getElementById('code').value;if(s)c=STDLIB+'\n'+c;const lines=[];const ol=console.log;console.log=(...a)=>lines.push(a.map(x=>String(x)).join(' '));try{const l=new Monkey.Lexer(c);const p=new Monkey.Parser(l);const pr=p.parseProgram();if(p.errors.length>0){o.textContent='Parse errors:\n'+p.errors.join('\n');console.log=ol;return}const co=new Monkey.Compiler();co.compile(pr);const vm=new Monkey.VM(co.bytecode());if(e==='jit')vm.enableJIT();const t=performance.now();vm.run();const el=performance.now()-t;const r=vm.lastPoppedStackElem();if(r&&r!==Monkey.NULL)lines.push('→ '+r.inspect());const tc=e==='jit'&&vm.jit?vm.jit.traces.size:0;document.getElementById('stats').textContent=el.toFixed(2)+'ms'+(tc>0?' | '+tc+' trace'+(tc>1?'s':''):'');o.textContent=lines.join('\n')||'(no output)'}catch(e){o.textContent='Error: '+e.message}console.log=ol}
function benchCode(){const o=document.getElementById('output');o.textContent='Benchmarking...';const s=document.getElementById('stdlib').checked;let c=document.getElementById('code').value;if(s)c=STDLIB+'\n'+c;const ol=console.log;console.log=()=>{};const N=50;let vt=0,jt=0;try{for(let i=0;i<N;i++){const l=new Monkey.Lexer(c);const p=new Monkey.Parser(l);const pr=p.parseProgram();const co=new Monkey.Compiler();co.compile(pr);const vm=new Monkey.VM(co.bytecode());const s=performance.now();vm.run();vt+=performance.now()-s}for(let i=0;i<N;i++){const l=new Monkey.Lexer(c);const p=new Monkey.Parser(l);const pr=p.parseProgram();const co=new Monkey.Compiler();co.compile(pr);const vm=new Monkey.VM(co.bytecode());vm.enableJIT();const s=performance.now();vm.run();jt+=performance.now()-s}const va=vt/N,ja=jt/N,sp=va/ja;o.textContent='Benchmark ('+N+' iterations)\n\n  VM:      '+va.toFixed(3)+'ms avg\n  JIT:     '+ja.toFixed(3)+'ms avg\n  Speedup: '+sp.toFixed(2)+'x'}catch(e){o.textContent='Error: '+e.message}console.log=ol}
function clearOutput(){document.getElementById('output').textContent='';document.getElementById('stats').textContent=''}
function loadExample(){const s=document.getElementById('examples');if(EXAMPLES[s.value])document.getElementById('code').value=EXAMPLES[s.value];s.value=''}
document.getElementById('code').addEventListener('keydown',e=>{if((e.ctrlKey||e.metaKey)&&e.key==='Enter'){e.preventDefault();runCode()}if(e.key==='Tab'){e.preventDefault();const t=e.target,s=t.selectionStart;t.value=t.value.substring(0,s)+'  '+t.value.substring(t.selectionEnd);t.selectionStart=t.selectionEnd=s+2}});
</script>
