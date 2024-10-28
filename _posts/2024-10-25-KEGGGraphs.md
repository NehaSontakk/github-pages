---
title: Generating KO (protein) graphs from string definitions via the KEGG API 
date: 2024-10-25
---

## Generating Tokens from Our String

Our first step is to parse a complex string that contains KEGG Orthology (KO) identifiers and special symbols representing hierarchies and relationships. The string we have is:

```
(((K01657+K01658,K13503,K13501,K01656) K00766),K13497) (((K01817,K24017) (K01656,K01609)),K13498,K13501) (K01695+(K01696,K06001),K01694)
```

### Understanding the Tokens

In this string, tokens are either:

- **KO identifiers**: Alphanumeric strings like `K01657`, `K13503`, etc.
- **Special symbols**: Characters that represent relationships, such as `(`, `)`, `+`, `,`, and spaces.

Our goal is to break down this string into these individual tokens for easier processing.

### Tokenization Strategy

To extract the tokens, we'll iterate over each character in the string and categorize it:

- **Alphanumeric Characters**: When we encounter an alphanumeric character, we'll keep collecting subsequent alphanumeric characters to form a complete KO identifier.
- **Special Symbols**: If we encounter one of the special symbols (`+`, `,`, `(`, `)`, or space), we'll treat it as a separate token.
- **Others**: Any other characters will be ignored as they don't represent meaningful tokens in our context.

## The `tokenize` Function

Here's a Python function that implements this logic:

```python
def tokenize(input_string):
    """
    Splits the input string into tokens.
    Tokens can be alphanumeric strings (KO identifiers) or symbols: '+', ',', '(', ')', ' '.
    """
    tokens = []
    i = 0
    while i < len(input_string):
        c = input_string[i]
        if c.isalnum():
            # Start collecting a KO identifier
            start = i
            while i < len(input_string) and input_string[i].isalnum():
                i += 1
            tokens.append(input_string[start:i])
        elif c in "+,() ":
            # It's a special symbol
            tokens.append(c)
            i += 1
        else:
            # Ignore unrecognized characters
            i += 1
    return tokens
```

## Applying the Function

Let's use the `tokenize` function on our string:

```python
input_string = "(((K01657+K01658,K13503,K13501,K01656) K00766),K13497) (((K01817,K24017) (K01656,K01609)),K13498,K13501) (K01695+(K01696,K06001),K01694)"
tokens = tokenize(input_string)
print(tokens)
```

**Output:**

```
['(',
 '(',
 '(',
 'K01657',
 '+',
 'K01658',
 ',',
 'K13503',
 ',',
 'K13501',
 ',',
 'K01656',
 ')',
 ' ',
 'K00766',
 ')',
 ',',
 'K13497',
 ')',
 ' ',
 '(',
 '(',
 '(',
 'K01817',
 ',',
 'K24017',
 ')',
 ' ',
 '(',
 'K01656',
 ',',
 'K01609',
 ')',
 ')',
 ',',
 'K13498',
 ',',
 'K13501',
 ')',
 ' ',
 '(',
 'K01695',
 '+',
 '(',
 'K01696',
 ',',
 'K06001',
 ')',
 ',',
 'K01694',
 ')']
```
