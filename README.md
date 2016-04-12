Kite is an artificial pair programmer, which helps you code better and faster. To allow Kite to do so, we've made various editor plugins to connect your programming environment to our backend. What's even better is that we are making all the source code for the plugins public, and we invite all hackers out there to make Kite alive for their favorite editors!

## Contributing
Kite currently supports `vim`, `Sublime Text`, `PyCharm` and `Atom`, and we expect the list to grow fast! To live the philosophy of *bring-your-own* editor, we made it extremely easy to create an editor plugin for Kite. Here we provide the guideline of how to do so. Join us to bring Kite to many more editors! Don't forget to also check out [CONTRIBUTING](https://github.com/kiteco/plugins/blob/master/CONTRIBUTING.md) to read more about how to contribute to Kite plugins.

## How to write a Kite plugin ##
The plugin needs to do two tihngs:
- Sending editor events to Kite.
- Receiving diffs and applying them to the editor. 

We will now go through how to accomplish these two tasks in more detail.

### Sending events to Kite ###
To support sending events to Kite:

1. Be able to grab:
 - Current buffer contents
 - Full filename
 - Current cursor position
 - Any selections (if they exist)

2. Be able to distinguish between an "edit" or "selection" event:
 - An "edit" event is a change in buffer contents. It need not be a single character change. 
 - A selection event is when the cursor is moved, or selection is changed without content being edited. Note: Sometimes editors will trigger an edit AND selection for regular typing (b/c the buffer was modified AND the cursor moved as a result of entering text). Its OK to send both of these events, `libkited` will correctly dedupe and send what is needed.
 - The cursor position is represented as a selection with `start == end` (see below).
 - These events should only be sent for the currently active buffer. Changes made to a non-active file (e.g. through multi-file find/replace) should not be sent.

3. The ability to write json blobs to a unix domain socket:
 - `libkited` listens to json objects sent to `$HOME/.kite/kite.sock`
 - A typical example of json structure being sent by vim is:
```javascript
{ 
  "source": "vim",
  "action": "edit", # could be "selection",
  "text": <buffer contents>,
  "selections": [{start: 5, end: 5}, {start: 10, end: 20}...],
}
```

**Note**: the `source` field should identify the particular editor. It's okay if multiple processes are open that use the same `source`. For PyCharm (which is a fork of IntelliJ targeting Python programmers) we still use "intellij" because it's the same code base as other IntelliJ forks or IntelliJ itself.

**Note 2**: filename's should be after resolving symlinks, resolving `.` and `..`, and getting to the underlying, canonical file path. In Java you can do this with `getCanonicalPath()`. In Python you can do this with `os.realpath`.

**Note 3**: before sending an event, check if the `text` is longer than 2^20 (1024*1024) characters. If it is, replace the event's action with "skip" and its contents with "file_too_large".

Here's a quick example of how this looks in python - it may vary based on the language you are working on for the plugin.
```py
SOCK_PATH = os.path.expandvars("$HOME/.kite/kite.sock")
uds = socket.socket(socket.AF_UNIX, socket.SOCK_DGRAM)
uds.setsockopt(socket.SOL_SOCKET, socket.SO_SNDBUF, 2<<20) # 2mb socket buffer
uds.sendto(json_body, SOCK_PATH)
```

### Receiving diffs ###
To receive diffs, you need to create and listen on a new unix domain socket:

- This unix domain socket must live in `$HOME/.kite/plugin_socks/`. 
- If your editor manages instances (e.g Sublime Text, when there's only ever one instance), you can use a socket name such as "sublime3". If your editor can have multiple instances (e.g `vim` or `emacs`), then you need to create a unique name. For `vim`, we simply use a `uuid` generator, and create sockets with the name `vim-<uuid>`.
- For any outgoing events you send on `$HOME/.kite/kite.sock`, you must add a new field: `pluginId`, set to the name of the unix domain socket you created.
- Be sure to remove this socket on editor close.

How this works: If a diff is generated by an event, the backend will copy the `pluginId` back into the response sent back to `libkited` which sends it along to the UI. If the user clicks on "apply", the sidebar contacts an endpoint at `/clientapi/python/applydiff` hosted by `libkited`, sending along the contents of the diff suggestion object. `libkited` is able to find your unix domain socket in `$HOME/.kite/plugin_socks` by name (via `pluginId`) and send the diff suggestion object to it. The diff suggestion object has the format:

```javascript
{
  "type": # either apply, highlight, clear>,
  "score": # backend score, unused,
  "plugin_id": # plugin_id,
  "file_md5": # md5 of file as it was when this suggestion was computed (lowercase hex string),
  "file_base64": # base64 encoded version of file when suggestion was computed,
  "diffs": <array of diff objects>,
}
```

The `file_md5` and `file_base64` fields assume a utf-8 encoding of the file contents.

Diff objects:
```javascript
{ 
  "type": "missing_import" or "typo",
  "linenum": # line number of the change,
  "begin": # byte offset to start of change,
  "end": # byte offset to end of change,
  "source": # what content looks like now,
  "destination": # what the content should look like,
  "line_src": # line number of the original contents
  "line_dst": # line number of the new contents,
}
```

Application of these diffs, and keeping track of offset changes as you apply a series of diff objects that may be part of the suggestion is up to the plugin implementation. 

After each apply, editor plugins should immediately execute an automatic "clear".

A "clear" event means: clear every highlight that the plugin has added in the past. You should probably therefore not use the `diffs` supplied with this event. Strictly speaking there could have been more things highlighted than are in the `diffs` field.

### Preamble

Each plugin source file should begin with the following comment:
```python
# Contents of this plugin will be reset by Kite on start. Changes you make
# are not guaranteed to persist.
```

### Some details

#### Character vs byte offsets

All offsets and lengths related to strings are specified in units of characters (i.e., unicode code points), not bytes (relevant for handling unicode strings).

#### Focus events

Events with an action called `focus` should be sent every time the user changes the currently-focused file in the editor, AND every time the editor window gains OS X focus.

Events with an action called `lost_focus` should be sent every time the editor window loses OS X focus.

Raising an event when the window gains OS X focus can be impossible with terminal editors. For example, [this discussion](http://apple.stackexchange.com/a/44377) seems to indicate decisively that there is no reliable vim hook for this. The draft plan is currently to use the accessibility API to track when terminal tabs get focus, and use a heuristic to match them to any running vim editor (still a todo).

#### Same file open multiple times

Some editors synchronize multiple buffers if they're for the same file, even without saves, e.g. IntelliJ, Eclipse, while other editors do not, e.g. Sublime Text.

The big saving grace here are the `focus` events. Without them and their repeat of the buffer contents, we'd have more trouble with this corner case.


## Authors
- `vim:` [Tarak Upadhyaya](https://github.com/tarakju), [Alex Flint](https://github.com/alexflint)
- `Sublime Text:` [Tarak Upadhyaya](https://github.com/tarakju), [Alex Flint](https://github.com/alexflint)
- `PyCharm:` [Adam Smith](https://github.com/adamsmith)
- `Atom:` [Tarak Upadhyaya](https://github.com/tarakju), [Alex Flint](https://github.com/alexflint)

## License
All the software in this repository is released under the MIT License. See [LICENSE](https://github.com/kiteco/plugins/blob/master/LICENSE) for details.
