# Jailpy3 & Kaizo Brackeys (LIT 2025)

I usually don't solve rev challenges (games are an exception) but I have no idea why a python jail was also put into the rev category...

## Jailpy3

> Made with a burning passion for pyjails (i.e. creating insane payloads just to bypass some random condition), I turned everything in this python script into pyjail tech! Here's a program that's suppose to print a flag. But something seems to be getting in the way...

We were provided with a **11.5MB** python file... calling this a jail challenge would be wrong instead just obfuscated python code.&#x20;

{% code overflow="wrap" fullWidth="false" %}
```python
# The starting of obfuscated python file provided
print({}.__class__.__subclasses__()[2].copy.__builtins__[{}.__class__.__subclasses__()[2].copy.__builtins__[chr(1^2^32^64)+chr(8^32^64)+chr(2^16^32^64)]({}.__class__.__subclasses__()[2].copy.__builtins__[chr(1^2^4^8^16^64)+chr(1^2^4^8^16^64)+chr(1^8^32^64)+chr(1^4^8^32^64)+chr(16^32^64)+chr(1^2^4^8^32^64)+chr(2^16^32^64)+chr(4^16^32^64)+chr(1^2^4^8^16^64)+chr(1^2^4^8^16^64)](chr(1^2^16^32^64)+chr(1^4^16^32^64)+chr(2^32^64)+chr(16^32^64)+chr(2^16^32^64)+chr(1^2^4^8^32^64)+chr(1^2^32^64)+chr(1^4^32^64)+chr(1^2^16^32^64)+chr(1^2^16^32^64)).select.POLLIN^{}.__class__.__subclasses__()[2].copy.__builtins__[chr(1^2^4^8^16^64)+chr(1^2^4^8^16^64)+chr(1^8^32^64)+chr(1^4^8^32^64)+chr(16^32^64)+chr(1^2^4^8^32^64)+chr(2^16^32^64)+chr(4^16^32^64)+chr(1^2^4^8^16^64)+chr(1^2^4^8^16^64)](chr(1^2^16^32^64)+chr(1^4^16^32^64)+chr(2^32^64)+chr(16^32^64)+chr(2^16^32^64)+chr(1^2^4^8^32^64)+chr(1^2^32^64)+chr(1^4^32^64)+chr(1^2^16^32^64)+chr(1^2^16^32^64)).select.POLLPRI^{}...
```
{% endcode %}

Seeing this code the first thing that we should do is replace `chr(...)` with the actual contents. So I asked **ChatGPT** to quickly write a script for me which does this for me.&#x20;

```python
import ast
import operator
import re
import sys
from pathlib import Path

ALLOWED_BINOPS = {
    ast.BitXor: operator.xor,
    ast.BitOr: operator.or_,
    ast.BitAnd: operator.and_,
    ast.LShift: operator.lshift,
    ast.RShift: operator.rshift,
    ast.Add: operator.add,
    ast.Sub: operator.sub,
    ast.Mult: operator.mul,
    ast.FloorDiv: operator.floordiv,
    ast.Mod: operator.mod,
}
ALLOWED_UNARY = {ast.Invert: operator.invert, ast.UAdd: operator.pos, ast.USub: operator.neg}

def safe_eval_int(expr: str) -> int:
    """Evaluate a small integer expression safely."""
    node = ast.parse(expr, mode="eval")

    def _eval(n):
        if isinstance(n, ast.Expression):
            return _eval(n.body)
        if isinstance(n, ast.Constant) and isinstance(n.value, int):
            return n.value
        if isinstance(n, ast.UnaryOp) and type(n.op) in ALLOWED_UNARY:
            return ALLOWED_UNARY[type(n.op)](_eval(n.operand))
        if isinstance(n, ast.BinOp) and type(n.op) in ALLOWED_BINOPS:
            return ALLOWED_BINOPS[type(n.op)](_eval(n.left), _eval(n.right))
        # Allow nested parentheses via Tuple with one elt in some Python versions? Not needed here.
        raise ValueError(f"disallowed expression: {ast.dump(n, include_attributes=False)}")

    val = _eval(node)
    if not isinstance(val, int):
        raise ValueError("non-integer expression")
    return val

def decode_chr_sequence_to_str(exprs):
    chars = []
    for e in exprs:
        code = safe_eval_int(e)
        try:
            chars.append(chr(code))
        except ValueError:
            raise ValueError(f"chr() out of range: {code}")
    return "".join(chars)

def scan_and_replace_chr_concats(src: str):
    """
    Find spans like: chr(<e1>) + chr(<e2>) + ... and replace with a quoted string.
    Handles whitespace and newlines.
    """
    i, n = 0, len(src)
    replacements = []

    while i < n:
        if src.startswith("chr(", i):
            start = i
            exprs = []

            def read_paren(idx):
                # idx at start of '('
                depth, j = 1, idx + 1
                while j < n and depth:
                    c = src[j]
                    if c == "(":
                        depth += 1
                    elif c == ")":
                        depth -= 1
                    j += 1
                if depth != 0:
                    raise ValueError("unbalanced parentheses in chr(...) sequence")
                return j  # position after ')'

            # first chr(
            if src.startswith("chr(", i):
                # read first expr
                open_idx = i + 3  # at '('
                end_after_paren = read_paren(open_idx)
                exprs.append(src[open_idx + 1 : end_after_paren - 1])

                k = end_after_paren
                # consume sequences of + chr(
                while True:
                    # skip whitespace
                    while k < n and src[k].isspace():
                        k += 1
                    if k < n and src[k] == "+":
                        k += 1
                        while k < n and src[k].isspace():
                            k += 1
                        if k < n and src.startswith("chr(", k):
                            open2 = k + 3
                            end2 = read_paren(open2)
                            exprs.append(src[open2 + 1 : end2 - 1])
                            k = end2
                            continue
                    break

                if len(exprs) >= 2:  # only replace concatenations, not single chr(...)
                    try:
                        decoded = decode_chr_sequence_to_str(exprs)
                        replacements.append((start, k, repr(decoded)))
                        i = k
                        continue
                    except Exception:
                        # fall through and advance one char if evaluation fails
                        pass
        i += 1

    # Apply replacements from end to start.
    if not replacements:
        return src, 0
    out = []
    last = 0
    for s, e, rep in sorted(replacements, key=lambda t: t[0]):
        out.append(src[last:s])
        out.append(rep)
        last = e
    out.append(src[last:])
    new_src = "".join(out)
    return new_src, len(replacements)

def deobfuscate_text(src: str, max_passes: int = 8):
    """Run multiple passes to catch cascaded patterns."""
    total = 0
    for _ in range(max_passes):
        src, cnt = scan_and_replace_chr_concats(src)
        total += cnt
        if cnt == 0:
            break
    return src, total

def main():
    if len(sys.argv) < 2:
        print("usage: python deobfuscate_chr_concat.py <input.py> [output.py]")
        sys.exit(2)
    inp = Path(sys.argv[1])
    outp = Path(sys.argv[2]) if len(sys.argv) >= 3 else None
    text = inp.read_text(encoding="utf-8")
    new_text, num = deobfuscate_text(text)
    if outp:
        outp.write_text(new_text, encoding="utf-8")
        print(f"replaced {num} chr-concat sequences -> {outp}")
    else:
        print(new_text)

if __name__ == "__main__":
    main()

```

After running this script the output looked like (note that it has been truncated... it was still **2.9MB**):

{% code overflow="wrap" %}
```python
# Truncated output after running first deobfuscation script
import collections
print({}.__class__.__subclasses__()[2].copy.__builtins__[{}.__class__.__subclasses__()[2].copy.__builtins__['chr']({}.__class__.__subclasses__()[2].copy.__builtins__['__import__']('subprocess').select.POLLIN^{}.__class__.__subclasses__()[2].copy.__builtins__['__import__']('subprocess').select.POLLPRI^{}.__class__.__subclasses__()[2].copy.__builtins__['__import__']('subprocess').select.POLLNVAL^{}.__class__.__subclasses__()[2].copy.__builtins__['__import__']('subprocess').select.POLLRDNORM)+{}.__class__.__subclasses__()[2].copy.__builtins__['chr']({}...
```
{% endcode %}

Now we can notice that there is use of long unnecessary paths to call Python builtin functions and imports. So now once again I gave these instructions to **GPT** and it produced a functional script.

```python
import ast, operator, re, sys
from pathlib import Path
import subprocess as _subprocess  # for select.POLL* constants

# ---------- safe integer evaluator ----------
_BINOPS = {
    ast.BitXor: operator.xor, ast.BitOr: operator.or_, ast.BitAnd: operator.and_,
    ast.Add: operator.add, ast.Sub: operator.sub, ast.Mult: operator.mul,
    ast.LShift: operator.lshift, ast.RShift: operator.rshift,
    ast.FloorDiv: operator.floordiv, ast.Mod: operator.mod,
}
_UNARY = {ast.Invert: operator.invert, ast.UAdd: operator.pos, ast.USub: operator.neg}

# expose only allowed names (after normalization we keep POLL* names)
_ALLOWED_NAMES = {name: getattr(_subprocess.select, name)
                  for name in dir(_subprocess.select) if name.startswith("POLL")}

def _safe_eval_int(expr: str) -> int:
    node = ast.parse(expr, mode="eval")

    def ev(n):
        if isinstance(n, ast.Expression): return ev(n.body)
        if isinstance(n, ast.Constant) and isinstance(n.value, int): return n.value
        if isinstance(n, ast.UnaryOp) and type(n.op) in _UNARY: return _UNARY[type(n.op)](ev(n.operand))
        if isinstance(n, ast.BinOp) and type(n.op) in _BINOPS: return _BINOPS[type(n.op)](ev(n.left), ev(n.right))
        if isinstance(n, ast.Name) and n.id in _ALLOWED_NAMES: return _ALLOWED_NAMES[n.id]
        raise ValueError(f"disallowed expression: {ast.dump(n, include_attributes=False)}")
    v = ev(node)
    if not isinstance(v, int): raise ValueError("non-integer")
    return v

# ---------- decoding helpers ----------
def _decode_chr_concat(src: str):
    """Replace chr(expr)+chr(expr)+... with quoted string."""
    i, n = 0, len(src); reps = []
    def read_paren(j):
        depth, k = 1, j+1
        while k < n and depth:
            c = src[k]
            if c == "(": depth += 1
            elif c == ")": depth -= 1
            k += 1
        if depth: raise ValueError("unbalanced parens")
        return k

    while i < n:
        if src.startswith("chr(", i):
            start = i
            aopen = i + 3
            aend = read_paren(aopen)
            exprs = [src[aopen+1:aend-1]]
            k = aend
            while True:
                while k < n and src[k].isspace(): k += 1
                if k < n and src[k] == "+":
                    k += 1
                    while k < n and src[k].isspace(): k += 1
                    if src.startswith("chr(", k):
                        bopen = k + 3
                        bend = read_paren(bopen)
                        exprs.append(src[bopen+1:bend-1])
                        k = bend
                        continue
                break
            if len(exprs) >= 2:
                try:
                    s = "".join(chr(_safe_eval_int(e)) for e in exprs)
                    reps.append((start, k, repr(s)))
                    i = k; continue
                except Exception:
                    pass
        i += 1
    if not reps: return src, 0
    out, last = [], 0
    for s, e, rep in reps:
        out.append(src[last:s]); out.append(rep); last = e
    out.append(src[last:])
    return "".join(out), len(reps)

def _decode_single_chr(src: str):
    """Replace chr(<int-expr>) with a quoted one-character string where safe."""
    # Avoid touching when followed/preceded by + which _decode_chr_concat handles.
    pattern = re.compile(r"chr\(\s*([^\(\)]*?)\s*\)")
    def repl(m):
        expr = m.group(1)
        # skip if contains quotes or disallowed chars to be safe
        if any(c in expr for c in "'\"[]{}"): return m.group(0)
        try:
            ch = chr(_safe_eval_int(expr))
            return repr(ch)
        except Exception:
            return m.group(0)
    new = pattern.sub(repl, src)
    changed = int(new != src)
    return new, changed

# ---------- normalization passes ----------
_BUILTINS_JAIL = re.compile(
    r"\{\}\.__class__\.__subclasses__\(\)\[\d+\]\.copy\.__builtins__"
)
def _normalize(src: str):
    s = src
    # 1) Collapse jail path to __builtins__
    s = _BUILTINS_JAIL.sub("__builtins__", s)
    # 2) Replace __builtins__['chr']( … ) -> chr( … )
    s = re.sub(r"__builtins__\s*\[\s*['\"]chr['\"]\s*\]\s*\(", "chr(", s)
    # 3) Replace __builtins__['__import__']('subprocess') -> subprocess
    s = re.sub(
        r"__builtins__\s*\[\s*['\"]__import__['\"]\s*\]\s*\(\s*['\"]subprocess['\"]\s*\)",
        "subprocess", s)
    # 4) Replace subprocess.select.POLLXXX -> POLLXXX
    s = re.sub(r"subprocess\s*\.\s*select\s*\.\s*(POLL[A-Z_]+)", r"\1", s)
    return s

def simplify_text(src: str, max_passes: int = 10):
    s = _normalize(src)
    total = 0
    for _ in range(max_passes):
        s2, n1 = _decode_chr_concat(s)
        s3, n2 = _decode_single_chr(s2)
        s4 = _normalize(s3)  # new opportunities may appear
        total += n1 + n2
        if s4 == s:
            break
        s = s4
    return s, total

def main():
    if len(sys.argv) < 2:
        print("usage: python simplify_ctf_obfuscation.py <input.py> [output.py]")
        sys.exit(2)
    inp = Path(sys.argv[1]); text = inp.read_text(encoding="utf-8")
    out, n = simplify_text(text)
    if len(sys.argv) >= 3:
        Path(sys.argv[2]).write_text(out, encoding="utf-8")
        print(f"simplified with {n} chr-rewrites -> {sys.argv[2]}")
    else:
        print(out)

if __name__ == "__main__":
    main()

```

Finally, after executing this deobfuscation script we got our **flag:**

```
LITCTF{h0w_c0nvolu7ed_c4n_i7_g37_f0r_0n3_s1mpl3_w0rk4round??}
```

{% code overflow="wrap" %}
```python
import collections
print('LITCTF{h0w_c0nvolu7ed_c4n_i7_g'__builtins__['__import__']('types').FunctionType(__builtins__['__import__']('marshal').loads(__builtins__['bytes'].fromhex('630000000000000000000000000300000000000000f3300000009700640064016c005a00020065006a020000000000000000000000000000000000006402ab01000000000000010079012903e9000000004ee9010000002902da026f73da055f65786974a900f300000000fa033c783efa083c6d6f64756c653e7208000000010000007314000000f003010101db0009883888328f38893890418d3b7206000000')), {'os': __builtins__['__import__']('os')})()+'37_f0r_0n3_s1mpl3_w0rk4round??}')

```
{% endcode %}

The challenge did look hard at start, probably I got scared because of the large file size, but by using AI to write the scripts the challenge simplified to two simple observations.&#x20;
