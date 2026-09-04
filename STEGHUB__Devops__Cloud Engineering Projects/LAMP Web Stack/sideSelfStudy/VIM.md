# VIM Text Editor Notebook

## Introduction to Vim

Vim (Vi Improved) is a powerful text editor designed for creating and modifying text efficiently from the keyboard. Unlike many modern editors that rely heavily on the mouse, Vim focuses on keyboard-based navigation and editing, allowing users to work quickly once they become familiar with its commands.

One of Vim's defining features is its use of different modes. Instead of treating every key press as text input, Vim assigns specific functions to keys depending on the mode currently in use. This approach makes editing, navigation, and text manipulation faster and more efficient.

---

# Vim Modes

Understanding Vim starts with understanding its modes. The two primary modes are **Normal Mode** and **Insert Mode**, although Vim also provides **Visual Mode** for text selection.

| Mode        | Purpose                                           | How to Enter |
| ----------- | ------------------------------------------------- | ------------ |
| Normal Mode | Used for navigation and editing commands          | Press `Esc`  |
| Insert Mode | Used for typing and inserting text                | Press `i`    |
| Visual Mode | Used for selecting text before performing actions | Press `v`    |

The current mode can usually be seen in Vim's status area.

---

# Switching Between Modes

Moving between modes is straightforward:

| Action                | Key   |
| --------------------- | ----- |
| Enter Insert Mode     | `i`   |
| Return to Normal Mode | `Esc` |
| Enter Visual Mode     | `v`   |

For example, if you are writing text and want to issue commands, press `Esc` to return to Normal Mode.

---

# Navigating Through Text

One of Vim's strengths is its ability to move through text quickly without using arrow keys.

| Key | Function                                   | Mode   |
| --- | ------------------------------------------ | ------ |
| `w` | Move to the beginning of the next word     | Normal |
| `e` | Move to the end of the current word        | Normal |
| `b` | Move to the beginning of the previous word | Normal |
| `0` | Move to the start of the line              | Normal |
| `$` | Move to the end of the line                | Normal |

These commands can be combined with numbers to move multiple words at once.

### Example

| Command | Result                   |
| ------- | ------------------------ |
| `3w`    | Move forward three words |
| `2b`    | Move back two words      |

---

# Finding Characters and Matching Symbols

Vim provides quick ways to locate characters and symbols within text.

| Key | Function                                    | Mode   |
| --- | ------------------------------------------- | ------ |
| `f` | Find the next occurrence of a character     | Normal |
| `F` | Find the previous occurrence of a character | Normal |
| `%` | Jump to the matching bracket or parenthesis | Normal |

### Example

| Command | Result                                     |
| ------- | ------------------------------------------ |
| `fo`    | Moves the cursor to the next letter "o"    |
| `%`     | Jumps between matching `()`, `{}`, or `[]` |

---

# Searching Within a File

Searching is essential when working with large files.

| Key     | Function                                                        | Mode   |
| ------- | --------------------------------------------------------------- | ------ |
| `/text` | Search for text                                                 | Normal |
| `n`     | Move to the next search result                                  | Normal |
| `N`     | Move to the previous search result                              | Normal |
| `*`     | Search for the next occurrence of the word under the cursor     | Normal |
| `#`     | Search for the previous occurrence of the word under the cursor | Normal |

---

# Moving Around the File

Navigating an entire file can be done with a few simple commands.

| Key               | Function                        | Mode   |
| ----------------- | ------------------------------- | ------ |
| `gg`              | Go to the beginning of the file | Normal |
| `G`               | Go to the end of the file       | Normal |
| `line_number + G` | Jump to a specific line         | Normal |

### Example

`25G` moves directly to line 25.

---

# Inserting Text

Besides the standard insert mode, Vim can create new lines automatically.

| Key | Function                                      | Mode   |
| --- | --------------------------------------------- | ------ |
| `i` | Insert text at cursor position                | Normal |
| `o` | Create a new line below and enter Insert Mode | Normal |
| `O` | Create a new line above and enter Insert Mode | Normal |

### Repeating Text

Text can be inserted multiple times using a count before entering Insert Mode.

Example:

`3iHello<Esc>`

This inserts "Hello" three times.

---

# Editing and Replacing Text

Vim provides several commands for modifying text efficiently.

| Key | Function                       | Mode   |
| --- | ------------------------------ | ------ |
| `x` | Delete character under cursor  | Normal |
| `X` | Delete character before cursor | Normal |
| `r` | Replace a single character     | Normal |

### Example

If the cursor is on the letter "a":

`rb`

The letter "a" is immediately replaced with "b".

---

# Deleting Text

Deletion commands become more powerful when combined with movement commands.

| Command | Function               | Mode   |
| ------- | ---------------------- | ------ |
| `d`     | Delete text            | Normal |
| `dw`    | Delete one word ahead  | Normal |
| `d2w`   | Delete two words ahead | Normal |

In standard Vim, deleted text is also copied into memory and can be pasted elsewhere.

| Key | Function                     |
| --- | ---------------------------- |
| `p` | Paste deleted or copied text |

---

# Repeating Commands

One of Vim's most useful features is command repetition.

| Key | Function                | Mode   |
| --- | ----------------------- | ------ |
| `.` | Repeat the last command | Normal |

### Example

After using:

`d2w`

Pressing `.` performs the same deletion again.

---

# Visual Mode

Visual Mode allows users to highlight text before applying an action.

| Key | Function                              | Mode   |
| --- | ------------------------------------- | ------ |
| `v` | Enter Visual Mode                     | Normal |
| `e` | Extend selection to the end of a word | Visual |
| `d` | Delete selected text                  | Visual |

### Example

1. Press `v`
2. Press `e`
3. Press `d`

The selected word is deleted.

---

# Saving and Exiting Vim

The following commands are among the most important to remember.

| Command | Function            |
| ------- | ------------------- |
| `:w`    | Save file           |
| `:q`    | Quit Vim            |
| `:q!`   | Quit without saving |

---

# Undo and Redo

Mistakes are inevitable, and Vim provides simple recovery commands.

| Key        | Function                      |
| ---------- | ----------------------------- |
| `u`        | Undo last change              |
| `Ctrl + R` | Redo previously undone change |

---

# Getting Help

Vim includes extensive built-in documentation.

| Command | Function             |
| ------- | -------------------- |
| `:help` | Open Vim help system |

Whenever you are unsure about a command or want to learn additional features, the help system is a valuable resource.

---

# Conclusion

Vim may feel unusual at first because of its mode-based design, but that design is what makes it so efficient. By mastering a few essential commands for navigation, editing, searching, and file management, users can perform many tasks without ever taking their hands off the keyboard. With regular practice, Vim becomes a fast and productive tool for editing text and managing code.


## LEARNING PROCESS AND PRACTISE

![VIM](<..\IMAGES\Screenshot 2026-06-10 162610.png>)

![VIM](<..\IMAGES\Screenshot 2026-06-10 164811.png>)

![VIM](..\IMAGES\FF.png)

![VIM](..\IMAGES\PRAC1.png)
