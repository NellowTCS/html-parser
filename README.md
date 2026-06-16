# html_parser

A fast, streaming, zero-dependency HTML token scanner written in ANSI C. 

This parser processes HTML text in place with an emphasis on raw performance and predictable memory overhead. It is designed to be highly resilient, successfully recovering from malformed markup, unclosed quotes, and irregular entity formatting just like a modern web browser.

---

## Backstory™️

As is tradition with a few of my projects, this codebase was written, buried, and completely forgotten about for several months before being rediscovered uncommitted in a local directory. It has now been rescued, open-sourced under the MIT license, and given the documentation it deserves. The implementation is heavily inspired by wget's `html-parse`, but no code was copied over; all code was implemented myself.

---

## Technical Stuff & Architecture

### 1. Stack-First Memory Arena (`Pool`)
Most HTML tags have short names and very few attributes. To eliminate the overhead of thousands of tiny heap allocations, `html_parser` utilizes a hybrid stack-to-heap string pool. 
* It initializes using a small buffer directly on the CPU stack.
* If a massive tag or attribute list threatens to overflow the stack buffer, it cleanly escalates to a heap-allocated buffer and scales exponentially via `realloc`.
* For 95% or more of typical web markup, zero heap allocations occur.

### 2. Stream-Optimized Processing
The parser does not build a heavy DOM tree. Instead, it streams through the buffer sequentially, invoking a user-defined callback every time a tag opens, closes, or carries attributes.

### 3. Wget-Inspired Resilience
The core parsing states, strict comment handling, and recovery boundary rules are heavily inspired by the robust HTML parsing engine found deep within GNU Wget. It handles real-world web phenomena seamlessly:
* **Boyer-Moore-ish Comment Skipping:** Uses a high-speed pointer jump strategy (`p += 3`) to scan past comments quickly.
* **Smart Backouting:** If it encounters a standalone `<` or an empty unquoted value (like `<foo bar=>`), it backtracks gracefully and treats the delimiter as literal text rather than crashing or breaking the state machine.

---

## Usage

The implementation is designed to be a frictionless drop-in utility.

### Standalone Testing
You can compile and run the parser as a self-contained binary with zero external dependencies by defining the `HTML_PARSER_STANDALONE` macro:

```bash
# Compile the standalone driver
cc -DHTML_PARSER_STANDALONE -o test html_parser.c

# Feed it some HTML
echo "<div class='container'><p>Hello World</p></div>" | ./test

```

### API Integration

To integrate it into your own C/C++ engine, simply define your callback and invoke `html_scan_tags`:

```c
#include "html_parser.h"

void my_tag_callback(struct html_tag *tag, void *user_data) {
    printf("Tag: %s (Is End Tag: %s)\n", tag->name, tag->is_end_tag ? "true" : "false");
    
    for (int i = 0; i < tag->nattrs; ++i) {
        printf("  Attribute: %s = %s\n", tag->attrs[i].name, tag->attrs[i].value);
    }
}

int main(void) {
    const char *html = "<a href='[https://github.com](https://github.com)'>GitHub</a>";
    int len = strlen(html);

    // Scan the buffer
    html_scan_tags(html, len, my_tag_callback, NULL, 0, NULL, NULL);
    
    return 0;
}
```

---

## License

This project is licensed under the MIT License. Feel free to modify, drop into your own software, or reuse it however you like.

```

```
