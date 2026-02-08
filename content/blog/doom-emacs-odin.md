+++
title = "Doom Emacs setup for Odin"
date = 2026-02-08
tags = ["Odin"]
draft = false
+++

Doom Emacs, as expected, can be made into great "IDE" for Odin.

This post will cover setting up syntax highlighting, LSP, build command, and debugger.


## Syntax highlighting {#syntax-highlighting}

Odin Tree-sitter mode provides a nice syntax highlighting.

Add the following to `packages.el`:

```elisp
;; packages.el

(package! odin-ts-mode
  :recipe (:host github :repo "Sampie159/odin-ts-mode"))
```

And `config.el`:

```elisp
;; config.el

(use-package! odin-ts-mode
  :mode "\\.odin\\'")
```

Odin Tree-sitter grammar needs to be installed separately.

Add the following to `config.el`:

```elisp
;; config.el

(after! treesit
  (add-to-list 'treesit-language-source-alist
               '(odin "https://github.com/tree-sitter-grammars/tree-sitter-odin")))
```

Syncronize your config with Doom Emacs (`doom sync`).


## LSP {#lsp}

Enable LSP support in `init.el` (I use Eglot, but LSP-mode also works fine):

```elisp
;; init.el

(lsp +eglot +peek)
;; (lsp +peek) ;; LSP-mode
```

Doom Emacs should be able to automatically install Odin's LSP server **ols**, but unfortunatelly it doesn't work for me.<br />
I use ols installed from sources:

1.  Clone <https://github.com/DanielGavin/ols> to ~/opt/ols (you can use any other path, but note that ~/opt/ols is used in this guide)
2.  Follow the build instructions to build both ols and odinfmt
3.  Add ~/opt/ols to $PATH (so that ols and odinfmt can be accessed as just `ols` / `odinfmt`)

Add hook to start Eglot when openinig Odin file and set up ols:

```elisp
;; config.el

(add-hook 'odin-ts-mode-hook #'eglot-ensure)

(setq lsp-odin-ols-binary-path "~/opt/ols/ols")
```

Syncronize your config with Doom Emacs (`doom sync`).


## Build command {#build-command}

While editing any Odin file, it should be possible to easily call `odin build` without switching to terminal.<br />
Emacs provides compile/recompile commands that can be conveniently used for this.

To have a correct compile command (that may include various build options) the compile/recompile commands can work with, set up [Just](https://github.com/casey/just) command runner.

Create `justfile`:

```nil
# justfile

default: build

build:
    odin build . -debug -out:out
```

And create `.dir-locals.el`:

```elisp
;; .dir-locals.el

(
 (nil . ((eval . (setq-local compile-command "just"))))
)
```

Now, after you open a directory, Emacs will load the directory's `.dir-locals.el` and update compile command.<br />
It'll be possible to build the project using `SPC c c` / `SPC c C` from any Odin file (including from subdirectories).


### Shell script instead of Just {#shell-script-instead-of-just}

If you don't want to use Just - you can create a simple `build.sh`:

```shell
#!/bin/sh

SCRIPT_DIR=$(cd "$(dirname "$0")" && pwd) || exit 1
cd "$SCRIPT_DIR" || exit 1

odin build . -debug -out:out
```

And then `.dir-locals.el`:

```elisp
;; .dir-locals.el

(
 (nil . ((eval . (setq-local compile-command
                             (concat (locate-dominating-file
                                      default-directory ".dir-locals.el") "build.sh")))))
)
```

However, Just is convenient as you'll likely need more commands for your development purposes anyways.


## Debugger {#debugger}

Install latest llvm and make sure `lldb-dap` is accessible.

Also, enable `debugger` in `init.el`:

```elisp
;; init.el

:tools
debugger
```

Modify `.dir-locals.el` added in the previos section:

```elisp
;; .dir-locals.el

(
  (nil . ((eval . (setq-local compile-command "just"))))
  (odin-ts-mode
  . ((eval
      . (let ((project-root
               (expand-file-name
                (file-name-as-directory
                 (file-name-directory
                  (locate-dominating-file
                   default-directory ".dir-locals.el"))))))
          (add-to-list
           'dape-configs
           `(odin-debug
             modes (odin-ts-mode)
             ensure dape-ensure-command
             command "lldb-dap"
             command-cwd dape-cwd-fn
             :type "lldb"
             :request "launch"
             :name "Odin Debug"
             :program ,(concat project-root "out")
             :cwd ,project-root
             :args []
             :stopOnEntry nil
             ;; :preRunCommands [,(concat "command script import " (expand-file-name "~/") "opt/odin-lldb/odin_lldb.py")]
             ))))))

)
```

Reopen any Odin file so that Emacs reloads `.dir-locals.el`.

Now, you can use debugger: add some breakpoints and run debugger (`SPC d d`).<br />
You should see "Run adapter: odin-debug", select it and wait for your breakpoints being hit.


### LLDB formatters {#lldb-formatters}

By default, lldb doesn't know how to format Odin's types (strings, slices etc.).

Notice the commented out line in `.dir-locals.el` above:

```elisp
;; :preRunCommands [,(concat "command script import " (expand-file-name "~/") "opt/odin-lldb/odin_lldb.py")]
```

It's possible to extend lldb to format Odin's types properly.<br />
Uncomment the line and add ~/opt/odin-lldb/odin_lldb.py:

```python
# odin_lldb.py

import lldb


class OdinSliceSyntheticProvider:
    def __init__(self, valobj, internal_dict):
        self.valobj = valobj
        self._data_ptr = None
        self._length = 0
        self._element_type = None
        self._element_size = 0

    def num_children(self):
        return self._length

    def get_child_index(self, name):
        try:
            return int(name.lstrip("[").rstrip("]"))
        except ValueError:
            return -1

    def get_child_at_index(self, index):
        if index < 0 or index >= self._length:
            return None
        if self._data_ptr is None or self._element_type is None:
            return None

        offset = index * self._element_size
        addr = self._data_ptr + offset
        return self.valobj.CreateValueFromAddress(
            f"[{index}]", addr, self._element_type
        )

    def update(self):
        self._data_ptr = None
        self._length = 0
        self._element_type = None
        self._element_size = 0

        try:
            data = self.valobj.GetChildMemberWithName("data")
            length = self.valobj.GetChildMemberWithName("len")

            if not data.IsValid() or not length.IsValid():
                return

            self._length = length.GetValueAsUnsigned(0)
            self._data_ptr = data.GetValueAsUnsigned(0)

            if self._data_ptr == 0:
                self._length = 0
                return

            ptr_type = data.GetType()
            if ptr_type.IsPointerType():
                self._element_type = ptr_type.GetPointeeType()
                self._element_size = self._element_type.GetByteSize()

            # cap the number of children to avoid performance issues
            self._length = min(self._length, 1000)

        except Exception:
            pass

    def has_children(self):
        return self._length > 0


class OdinDynamicArraySyntheticProvider(OdinSliceSyntheticProvider):
    pass


def odin_string_summary(valobj, internal_dict):
    try:
        data = valobj.GetChildMemberWithName("data")
        length = valobj.GetChildMemberWithName("len")

        if not data.IsValid() or not length.IsValid():
            return None

        len_val = length.GetValueAsUnsigned(0)
        if len_val == 0:
            return '""'

        # limit display length
        display_len = min(len_val, 256)

        data_addr = data.GetValueAsUnsigned(0)
        if data_addr == 0:
            return "<nil>"

        error = lldb.SBError()
        process = valobj.GetProcess()
        content = process.ReadMemory(data_addr, display_len, error)

        if error.Fail():
            return None

        try:
            text = content.decode("utf-8", errors="replace")
        except Exception:
            text = repr(content)

        if len_val > display_len:
            return f'"{text}..." (len={len_val})'
        return f'"{text}"'

    except Exception:
        return None


def odin_slice_summary(valobj, internal_dict):
    try:
        valobj = valobj.GetNonSyntheticValue()
        data = valobj.GetChildMemberWithName("data")
        length = valobj.GetChildMemberWithName("len")

        if not data.IsValid() or not length.IsValid():
            return None

        len_val = length.GetValueAsUnsigned(0)
        data_addr = data.GetValueAsUnsigned(0)

        if data_addr == 0 and len_val == 0:
            return "[] (len=0)"
        if data_addr == 0:
            return f"<nil> (len={len_val})"

        return f"len={len_val}"

    except Exception:
        return None


def odin_dynamic_array_summary(valobj, internal_dict):
    try:
        valobj = valobj.GetNonSyntheticValue()
        data = valobj.GetChildMemberWithName("data")
        length = valobj.GetChildMemberWithName("len")
        cap = valobj.GetChildMemberWithName("cap")

        if not data.IsValid() or not length.IsValid():
            return None

        len_val = length.GetValueAsUnsigned(0)
        cap_val = cap.GetValueAsUnsigned(0) if cap.IsValid() else 0
        data_addr = data.GetValueAsUnsigned(0)

        if data_addr == 0 and len_val == 0:
            return f"[] (len=0, cap={cap_val})"
        if data_addr == 0:
            return f"<nil> (len={len_val}, cap={cap_val})"

        return f"len={len_val}, cap={cap_val}"

    except Exception:
        return None


def __lldb_init_module(debugger, internal_dict):
    debugger.HandleCommand(
        'type summary add -F odin_lldb.odin_string_summary "string" -w odin'
    )

    debugger.HandleCommand(
        'type summary add -x "^\\[\\].+" -F odin_lldb.odin_slice_summary -w odin'
    )
    debugger.HandleCommand(
        'type synthetic add -x "^\\[\\].+" -l odin_lldb.OdinSliceSyntheticProvider -w odin'
    )

    debugger.HandleCommand(
        'type summary add -x "^\\[dynamic\\].+" -F odin_lldb.odin_dynamic_array_summary -w odin'
    )
    debugger.HandleCommand(
        'type synthetic add -x "^\\[dynamic\\].+" -l odin_lldb.OdinDynamicArraySyntheticProvider -w odin'
    )

    debugger.HandleCommand("type category enable odin")

    print("Odin LLDB formatters loaded.")
```

It will extend lldb to properly format strings, dynamic arrays and slices (more types can be added e.g. maps etc.).<br />
For example, in the debugger you will see:

```nil
+ some_string "some_string" string
- some_array len=3, cap=8 [dynamic]string
  + [0] "one" string
  + [1] "two" string
  + [2] "three" string
- some_slice len=2 []string
  + [0] "one" string
  + [1] "two" string
```

instead of

```nil
- some_string string @ 0x16fdfe140 string
  + data 0x0000000100127ef4 "some_string" u8 *
    len 11 int
- some_array [dynamic]string @ 0x16fdfe0e0 [dynamic]string
  - data 0x0000000100c52068 string *
    + data 0x0000000100127f00 "one" u8 *
      len 3 int
    len 3 int
    cap 8 int
  + allocator runtime::Allocator @ 0x16fdfe0f8 runtime::Allocator
- some_slice []string @ 0x16fdfe050 []string
  - data 0x0000000100c52068 string *
    + data 0x0000000100127f00 "one" u8 *
      len 3 int
    len 2 int
```

The summaries (the most top level rows) are much more usefull, and arrays/slices have all the elements displayed instead of just the first one.


## Conclusion {#conclusion}

Doom Emacs can be made into nice "IDE" for Odin.

There're several rough edges, but the ecosystem should mature with time, and the setup process should be easier and more out-of-the-box than it is now.<br />
However, even now the setup process isn't too complicated, and it's a one time effort to get pretty great experience.
