# PostgreSQL SQL Lexer Modification Guide

## Overview

This guide provides essential information for developers who want to modify PostgreSQL's SQL lexer, which is responsible for breaking SQL queries into tokens for the parser. Understanding the lexer architecture, key files, and concepts will help you make effective changes to the tokenization process.

## Architecture

The PostgreSQL SQL lexer is implemented using Flex (a lexical analyzer generator) and works closely with the Bison-based parser. The main components are:

- **Lexer (scan.l)**: Defines token patterns using regular expressions
- **Parser (gram.y)**: Defines the grammar rules that consume tokens from the lexer
- **Parser Interface (parser.c)**: Provides API for invoking the parser/lexer
- **Support Functions (scansup.c)**: Helper functions for the lexer

The lexical analysis process follows these steps:
1. Raw SQL text is fed to the lexer
2. The lexer identifies tokens based on regular expression patterns
3. Each recognized token is passed to the parser
4. The parser builds a parse tree based on the token stream

## Key Files and Their Purposes

| File | Location | Description |
|------|----------|-------------|
| `scan.l` | `src/backend/parser/` | Main lexer definition written in Flex |
| `gram.y` | `src/backend/parser/` | Bison grammar that consumes lexer tokens |
| `parser.c` | `src/backend/parser/` | Interface between application code and parser/lexer |
| `scansup.c` | `src/backend/parser/` | Support functions for the lexer |
| `kwlist.h` | `src/include/parser/` | List of SQL keywords |
| `gramparse.h` | `src/backend/parser/` | Generated header defining token types |

## Core Concepts and APIs

### Lexer States

The lexer uses different states to handle context-sensitive tokenization:

```
/* Main states */
%x xb       /* bit string literal */
%x xc       /* extended C-style comments */
%x xd       /* delimited identifiers (double-quoted identifiers) */
%x xh       /* hexadecimal byte string */
%x xq       /* standard quoted strings */
%x xqs      /* quote stop (detect continued strings) */
%x xe       /* extended quoted strings (support backslash escape sequences) */
%x xdolq    /* $foo$ quoted strings */
%x xui      /* quoted identifier with Unicode escapes */
%x xus      /* quoted string with Unicode escapes */
```

### Token Structure

Tokens are defined in `gram.y` and their values are generated in `gramparse.h`:

```c
/* Sample token definitions from gram.y */
%token <keyword> SELECT INSERT UPDATE DELETE MERGE
%token <keyword> ABORT ABS ABSOLUTE ACCESS ACTION ADD ADMIN AFTER
/* ... more token definitions ... */
```

### Lexer Rules

Lexer rules in `scan.l` define patterns and actions for token recognition:

```
/* Example rule for identifiers */
{identifier}    {
                    int         kwnum;
                    char       *ident;

                    SET_YYLLOC();
                    /* ... processing code ... */
                    return IDENT;
                }
```

### Scanner Interface

The main function for invoking the lexer/parser is `raw_parser()` in `parser.c`:

```c
List *
raw_parser(const char *str, RawParseMode mode)
{
    /* Initialize scanner and parser */
    /* Parse input string */
    /* Return parse tree */
}
```

### Keywords Handling

Keywords are defined in `kwlist.h` and loaded by the lexer:

```c
#define PG_KEYWORD(kwname, value, category, collabel) value,

const uint16 ScanKeywordTokens[] = {
#include "parser/kwlist.h"
};
```

## Common Modification Patterns

### Adding a New SQL Keyword

1. Add the keyword to `src/include/parser/kwlist.h`
   ```c
   PG_KEYWORD("NEW_KEYWORD", NEW_KEYWORD, UNRESERVED_KEYWORD, "")
   ```

2. Define the token in `gram.y`
   ```c
   %token <keyword> NEW_KEYWORD
   ```

3. Add grammar rules in `gram.y` that use the new keyword

4. Update the documentation in `doc/src/sgml/sql-commands.sgml`

### Adding a New Token Type

1. Define the token in `gram.y`
   ```c
   %token <new_token_type> NEW_TOKEN
   ```

2. Add a pattern rule in `scan.l`
   ```
   new-pattern    {
                      SET_YYLLOC();
                      /* Process and return the new token */
                      return NEW_TOKEN;
                  }
   ```

3. Add any necessary actions for the token in the grammar file

### Modifying Existing Token Recognition

1. Locate the rule in `scan.l` that recognizes the token you want to modify
2. Adjust the regular expression pattern or the action code
3. Be careful with rule ordering as Flex chooses the longest matching pattern or the first matching pattern of equal length

### Adding a New Lexer State

1. Define the new state in `scan.l`
   ```
   %x xnewstate
   ```

2. Add rules for entering and exiting the state
   ```
   /* Enter the state */
   "trigger-pattern"    { BEGIN(xnewstate); /* other actions */ }
   
   /* Rules that apply in the new state */
   <xnewstate>rules...
   
   /* Exit the state */
   <xnewstate>"exit-pattern"    { BEGIN(INITIAL); /* other actions */ }
   ```

3. Don't forget to handle the EOF case for your new state

## Important Considerations

### No-Backtrack Property

PostgreSQL's lexer is designed to avoid backtracking for performance:

```
/* From scan.l header comment */
/* The rules are designed so that the scanner never has to backtrack...
 * this makes for a useful speed increase...
 * If you change the lexical rules, verify that you haven't broken the 
 * no-backtrack property by running flex with the "-b" option...
 */
```

When modifying rules, check that this property is maintained by running:
```bash
flex -b src/backend/parser/scan.l
```

### Synchronization with Frontend Lexers

The PostgreSQL backend lexer must be kept in sync with frontend lexers:

```
/* From scan.l header comment */
/* NOTE NOTE NOTE:
 * The rules in this file must be kept in sync with src/fe_utils/psqlscan.l
 * and src/interfaces/ecpg/preproc/pgc.l!
 */
```

After modifying `scan.l`, make corresponding changes to:
- `src/fe_utils/psqlscan.l` (for psql)
- `src/interfaces/ecpg/preproc/pgc.l` (for ECPG)

### Error Handling

Lexer errors are reported using `yyerror()` which maps to `scanner_yyerror()`:

```c
#define yyerror(msg)  scanner_yyerror(msg, yyscanner)
```

For position information in errors, use:

```c
#define lexer_errposition()  scanner_errposition(*(yylloc), yyscanner)
```

## Building and Testing

To rebuild the lexer after making changes:

```bash
cd src/backend/parser
make clean
make
```

To test your changes:

1. Rebuild PostgreSQL
   ```bash
   make
   make install
   ```

2. Test specific SQL statements that would exercise your lexer changes
   ```bash
   psql -c "SELECT your_modified_syntax;"
   ```

3. Run regression tests
   ```bash
   make check
   ```

## Common Pitfalls

- **Rule Order Matters**: Flex selects the rule that matches the longest input sequence; if there's a tie, the first rule in the file wins
- **State Management**: Always ensure proper state transitions, especially for error cases
- **Memory Management**: Be careful with memory allocation in lexer actions
- **Unicode Support**: Consider multi-byte character handling
- **Token Location Tracking**: Maintain proper location information for error reporting
- **Frontend Synchronization**: Keep frontend lexers in sync with backend changes
- **Performance Impact**: Lexer changes can significantly impact parsing performance

## Additional Resources

- PostgreSQL Developer Documentation: https://www.postgresql.org/docs/current/internals.html
- Flex Manual: https://westes.github.io/flex/manual/
- Bison Manual: https://www.gnu.org/software/bison/manual/
- PostgreSQL Mailing List: pgsql-hackers@lists.postgresql.org
- Example of a lexer change: https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=6127176
