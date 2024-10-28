---
title: Parsing KEGG module expressions to decipher KO (protein) relationships into graphical representations
date: 2024-10-25
---

I needed to generate accurate representations of metabolic pathways and protein interactions. While exploring the KEGG (Kyoto Encyclopedia of Genes and Genomes) database to reconstruct metabolic pathways, I was stuck trying to derive the relationships between KEGG Orthology (KO) identifiers from reaction and compound data alone. 

KEGG's reaction and compound graphs provide valuable information, but they don't directly depict complex KO relationships, especially those involving protein complexes and alternative pathways. For instance, proteins that function together as a complex are represented using '+' signs (e.g., K01657+K01658), and alternative pathways are indicated using commas. This detailed representation is not readily apparent or extractable from the reaction and compound data provided by the KEGG API.

I found that KEGG encapsulates their entire module graphs in the form of compact string expressions. These expressions concisely encode the complex relationships between KOs, including sequential reactions, protein complexes, and alternative pathways. For example, the expression for KEGG module M00023 is:

```
(((K01657+K01658,K13503,K13501,K01656) K00766),K13497) (((K01817,K24017) (K01656,K01609)),K13498,K13501) (K01695+(K01696,K06001),K01694)
```

Which is also represented like this:
![_posts/Pictures/2_ko_M00023.png](https://github.com/NehaSontakk/github-pages/blob/main/_posts/Pictures/2_ko_M00023.png)

This intricate expression represents a network of reactions and interactions that are challenging to reconstruct using reaction and compound data alone. After an unsuccessful search for an existing parser capable of interpreting these KEGG module expressions, I decided to develop one myself. _Disclaimer: I did this with the assistance of ChatGPT, and it was fabulous and pretty great at helping me construct a solution._

Here is the link to the complete code: ![KO_graph_from_expression.ipynb](https://github.com/NehaSontakk/KEGG_GRAPH/blob/abe6d4484e4d69e0aae0966617edea2db0955be4/KO_graph_from_expression.ipynb)

## Generating Tokens from Our String

Our first step is to parse a complex string that contains KEGG Orthology (KO) identifiers and special symbols representing hierarchies and relationships. The string we have is:

```
(((K01657+K01658,K13503,K13501,K01656) K00766),K13497) (((K01817,K24017) (K01656,K01609)),K13498,K13501) (K01695+(K01696,K06001),K01694)
```


### Understanding the tokens

In this string, tokens are either:

- **KO identifiers**: Alphanumeric strings like `K01657`, `K13503`, etc.
- **Special symbols**: Characters that represent relationships, such as `(`, `)`, `+`, `,`, and spaces.

Our goal is to break down this string into these individual tokens for easier processing.

### Tokenization strategy

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

## Parsing tokens as an expression

After tokenizing our complex string of KEGG Orthology (KO) identifiers and special symbols, the next step is to **parse the tokens** according to grammar rules to build a representation of the expression. This involves interpreting the hierarchical relationships represented by non-alphanumeric characters.

Think of this process as following a set of nested instructions or opening nested boxes. We use a series of **recursive functions** to handle different parts of the expression.

### Understanding the parsing strategy

Essentially, we're dealing with a mathematical-like expression and need to process it according to certain rules. While it's similar to the BODMAS (Brackets, Orders, Division/Multiplication, Addition/Subtraction) order of operations, our symbols have slightly different meanings:

- **Parentheses `(` and `)`**: Represent grouping of elements.
- **Plus `+`**: Represents an "AND" relationship.
- **Comma `,`**: Represents an "OR" relationship.
- **Space ` `**: Represents sequential steps.

### Parser functions

We define several parser functions to handle different parts of the expression:

- `parse_expr()`: Handles expressions separated by **spaces**.
- `parse_sum_expr()`: Handles expressions separated by **commas**.
- `parse_mult_expr()`: Handles expressions separated by **pluses**.
- `parse_factor()`: Handles individual elements or grouped expressions in **parentheses**.

These functions work together recursively to parse the entire expression.

### How the recursive functions work together

1. **`parse_expr`** starts by calling `parse_sum_expr` to handle the first part of the expression.
2. **`parse_sum_expr`** calls `parse_mult_expr`, which further calls `parse_factor`.
3. **`parse_factor`** checks if the current token is a `'('` to handle grouped expressions. If so, it recursively calls `parse_sum_expr` to parse the inner expression.
4. This process continues, with each function calling itself or another function as needed, until the entire expression is parsed.

### Actions to be performed
Create Nodes: For each KO identifier.
Create Edges: 'AND' Relationship: From one node to another when a '+' is encountered.
Sequential Steps: From last nodes of one expression to top nodes of the next when a space is encountered.

### Vizualization of the function calls for an example

I'll vizualize the function calls for functional block 1. Let's consider the full expression again:

```
(((K01657+K01658,K13503,K13501,K01656) K00766),K13497) (((K01817,K24017) (K01656,K01609)),K13498,K13501) (K01695+(K01696,K06001),K01694)
```

This expression can be broken down into **functional blocks** separated by spaces:

1. **Functional Block 1**:
   ```
   (((K01657+K01658,K13503,K13501,K01656) K00766),K13497)
   ```
2. **Functional Block 2**:
   ```
   (((K01817,K24017) (K01656,K01609)),K13498,K13501)
   ```
3. **Functional Block 3**:
   ```
   (K01695+(K01696,K06001),K01694)
   ```

These functional blocks represent steps or sequences that occur one after another. The spaces in the original expression separate these blocks.

### **Processing Functional Block 1 using our nested recursive functions**:

```
(((K01657+K01658,K13503,K13501,K01656) K00766),K13497)
```

**Tokens** (with indices):

| Token Index | Token    |
|-------------|----------|
| 0           | '('      |
| 1           | '('      |
| 2           | '('      |
| 3           | 'K01657' |
| 4           | '+'      |
| 5           | 'K01658' |
| 6           | ','      |
| 7           | 'K13503' |
| 8           | ','      |
| 9           | 'K13501' |
| 10          | ','      |
| 11          | 'K01656' |
| 12          | ')'      |
| 13          | ' '      |
| 14          | 'K00766' |
| 15          | ')'      |
| 16          | ','      |
| 17          | 'K13497' |
| 18          | ')'      |

---

### **Token-Function Mapping Table**

| Token Index | Token     | Function          | Action                                                                        |
|-------------|-----------|-------------------|-------------------------------------------------------------------------------|
| **0**       | `'('`     | `parse_expr`      | Call `parse_sum_expr()`                                                       |
| **1**       | `'('`     | `parse_sum_expr`  | Call `parse_mult_expr()`                                                      |
| **2**       | `'('`     | `parse_mult_expr` | Call `parse_factor()`                                                         |
| **2**       | `'('`     | `parse_factor`    | Detect `'('`, advance index, call `parse_sum_expr()`                          |
| **3**       | `'K01657'`| `parse_sum_expr`  | Call `parse_mult_expr()`                                                      |
| **3**       | `'K01657'`| `parse_mult_expr` | Call `parse_factor()`                                                         |
| **3**       | `'K01657'`| `parse_factor`    | Create node `'K01657'`, advance index to **4**                                |
| **4**       | `'+'`     | `parse_mult_expr` | Match `'+'`, advance index to **5**, call `parse_factor()`                    |
| **5**       | `'K01658'`| `parse_factor`    | Create node `'K01658'`, create edge `'K01657' → 'K01658'`, advance index to **6** |
| **6**       | `','`     | `parse_mult_expr` | No more `'+'`, return to `parse_sum_expr()`                                   |
| **6**       | `','`     | `parse_sum_expr`  | Match `','`, advance index to **7**, call `parse_mult_expr()`                 |
| **7**       | `'K13503'`| `parse_mult_expr` | Call `parse_factor()`                                                         |
| **7**       | `'K13503'`| `parse_factor`    | Create node `'K13503'`, advance index to **8**                                |
| **8**       | `','`     | `parse_mult_expr` | No more `'+'`, return to `parse_sum_expr()`                                   |
| **8**       | `','`     | `parse_sum_expr`  | Match `','`, advance index to **9**, call `parse_mult_expr()`                 |
| **9**       | `'K13501'`| `parse_mult_expr` | Call `parse_factor()`                                                         |
| **9**       | `'K13501'`| `parse_factor`    | Create node `'K13501'`, advance index to **10**                               |
| **10**      | `','`     | `parse_mult_expr` | No more `'+'`, return to `parse_sum_expr()`                                   |
| **10**      | `','`     | `parse_sum_expr`  | Match `','`, advance index to **11**, call `parse_mult_expr()`                |
| **11**      | `'K01656'`| `parse_mult_expr` | Call `parse_factor()`                                                         |
| **11**      | `'K01656'`| `parse_factor`    | Create node `'K01656'`, advance index to **12**                               |
| **12**      | `')'`     | `parse_mult_expr` | No more `'+'`, return to `parse_sum_expr()`                                   |
| **12**      | `')'`     | `parse_sum_expr`  | Expect `')'`, advance index to **13**, return to `parse_factor()`             |
| **13**      | `' '`     | `parse_factor`    | Return to `parse_mult_expr()`                                                 |
| **13**      | `' '`     | `parse_mult_expr` | Return to `parse_sum_expr()`                                                  |
| **13**      | `' '`     | `parse_sum_expr`  | No more `','`, return to `parse_factor()`                                     |
| **13**      | `' '`     | `parse_factor`    | Return to `parse_mult_expr()`                                                 |
| **13**      | `' '`     | `parse_mult_expr` | Return to `parse_sum_expr()`                                                  |
| **13**      | `' '`     | `parse_sum_expr`  | Return to `parse_expr()`                                                      |
| **13**      | `' '`     | `parse_expr`      | Match `' '`, advance index to **14**, call `parse_sum_expr()`                 |
| **14**      | `'K00766'`| `parse_sum_expr`  | Call `parse_mult_expr()`                                                      |
| **14**      | `'K00766'`| `parse_mult_expr` | Call `parse_factor()`                                                         |
| **14**      | `'K00766'`| `parse_factor`    | Create node `'K00766'`, create edges from previous nodes, advance index to **15** |
| **15**      | `')'`     | `parse_mult_expr` | No more `'+'`, return to `parse_sum_expr()`                                   |
| **15**      | `')'`     | `parse_sum_expr`  | Expect `')'`, advance index to **16**, return to `parse_factor()`             |
| **16**      | `','`     | `parse_factor`    | Return to `parse_mult_expr()`                                                 |
| **16**      | `','`     | `parse_mult_expr` | Return to `parse_sum_expr()`                                                  |
| **16**      | `','`     | `parse_sum_expr`  | Match `','`, advance index to **17**, call `parse_mult_expr()`                |
| **17**      | `'K13497'`| `parse_mult_expr` | Call `parse_factor()`                                                         |
| **17**      | `'K13497'`| `parse_factor`    | Create node `'K13497'`, advance index to **18**                               |
| **18**      | `')'`     | `parse_mult_expr` | No more `'+'`, return to `parse_sum_expr()`                                   |
| **18**      | `')'`     | `parse_sum_expr`  | Expect `')'`, advance index to **19**, return to `parse_expr()`               |

---

### Writing **`parse_expr`**, **`parse_sum_expr`**, **`parse_mult_expr`**, **`parse_factor`**


```python
tokens = []
current_token_index = 0
```

- **`tokens`**: The list of tokens generated by the `tokenize` function.
- **`current_token_index`**: Keeps track of the current position in the `tokens` list during parsing.

### Recursive Functions

#### `parse_expr`

```python
def parse_expr():
    """
    Parses an expression separated by spaces (' ').
    """
    top_nodes, last_nodes = parse_sum_expr()
    while match(" "):
        next_top_nodes, next_last_nodes = parse_sum_expr()
        # Create edges from last_nodes to next_top_nodes
        create_edges(last_nodes, next_top_nodes)
        last_nodes = next_last_nodes  # Update last nodes
    return top_nodes, last_nodes
```

- **Purpose**: Parses expressions separated by spaces, representing sequential steps.
- **Behavior**:
  - Calls `parse_sum_expr` to parse the first part.
  - While there's a space, it continues parsing the next expressions.
  - Creates edges from the `last_nodes` of the previous expression to the `top_nodes` of the next one.
  - Updates `last_nodes` to the `last_nodes` of the newly parsed expression.

#### `parse_sum_expr`

```python
def parse_sum_expr():
    """
    Parses an expression separated by commas (',').
    """
    top_nodes, last_nodes = parse_mult_expr()
    while match(","):
        next_top_nodes, next_last_nodes = parse_mult_expr()
        # Combine top_nodes and last_nodes
        top_nodes.extend(next_top_nodes)
        last_nodes.extend(next_last_nodes)
    return top_nodes, last_nodes
```

- **Purpose**: Parses expressions separated by commas, representing parallel paths.
- **Behavior**:
  - Parses the first multiplicative expression.
  - While there's a comma, it continues parsing additional multiplicative expressions.
  - Combines the `top_nodes` and `last_nodes` from each parsed expression.

#### `parse_mult_expr`

```python
def parse_mult_expr():
    """
    Parses an expression separated by pluses ('+').
    """
    top_nodes, last_nodes = parse_factor()
    while match("+"):
        next_top_nodes, next_last_nodes = parse_mult_expr()
        # Create edges from last_nodes to next_top_nodes
        create_edges(last_nodes, next_top_nodes)
        last_nodes = next_last_nodes  # Update last_nodes
    return top_nodes, last_nodes
```

- **Purpose**: Parses expressions separated by pluses, representing sequential steps without spaces.
- **Behavior**:
  - Parses a single factor.
  - While there's a '+', it continues parsing the next multiplicative expression.
  - Creates edges from the `last_nodes` of the previous expression to the `top_nodes` of the next one.
  - Updates `last_nodes` accordingly.

#### `parse_factor`

```python
def parse_factor():
    """
    Parses a single factor, which can be a token or a parenthesized expression.
    """
    if match("("):
        top_nodes, last_nodes = parse_expr()
        expect(")")
        return top_nodes, last_nodes
    elif current_token() is not None and current_token()[0].isalnum():
        token_name = current_token()
        consume_token()
        node = create_node(token_name)
        return [node], [node]
    else:
        raise Exception(f"Unexpected token: {current_token()}")
```

- **Purpose**: Parses a basic unit in the expression.
- **Behavior**:
  - If it encounters '(', it starts parsing a nested expression and expects a closing ')'.
  - If the current token is alphanumeric (a node name), it creates a new node.
  - If none of the above, it raises an exception.

---


### Helper functions
The recursive functions use helpers (refer to the code for definitions). 

 `current_token`: Retrieves the current token being analyzed.

 `match`: Checks if the current token matches a specific value. If it matches, it consumes the token (advances `current_token_index`) and returns `True`. If not, it returns `False`.

 `expect`: Ensures that the current token matches the expected value. If the token matches, continues parsing. If not, raises an exception with an error message.

`consume_token`: Advances to the next token without any checks.


### Node creation functions

#### `create_node`
Creates a new `Node` instance and adds it to the `nodes` list.

```python
def create_node(name):
    """
    Creates a new Node object with the given name.
    """
    node = Node(name)
    nodes.append(node)
    return node
```

#### `create_edges`
Creates edges between lists of nodes. For every node in `from_nodes`, it creates an edge to every node in `to_nodes`.

```python
def create_edges(from_nodes, to_nodes):
    """
    Creates edges from each node in from_nodes to each node in to_nodes.
    """
    for from_node in from_nodes:
        for to_node in to_nodes:
            edges.append((from_node, to_node))
```

### Building the graph

### `build_graph`

```python
def build_graph(input_string):
    """
    Main function to build the graph from the input string.
    """
    global tokens, current_token_index, nodes, edges
    tokens = tokenize(input_string)
    current_token_index = 0
    nodes = []
    edges = []
    parse_expr()

    # Add "source" and "sink" nodes
    source_node = Node("source")
    sink_node = Node("sink")
    nodes.extend([source_node, sink_node])

    # Determine nodes with no incoming edges
    node_incoming = {node: 0 for node in nodes}
    node_outgoing = {node: 0 for node in nodes}
    for from_node, to_node in edges:
        node_incoming[to_node] = node_incoming.get(to_node, 0) + 1
        node_outgoing[from_node] = node_outgoing.get(from_node, 0) + 1

    # Add edges from "source" to nodes with no incoming edges
    for node in nodes:
        if node_incoming.get(node, 0) == 0 and node != source_node and node != sink_node:
            edges.append((source_node, node))

    # Add edges from nodes with no outgoing edges to "sink"
    for node in nodes:
        if node_outgoing.get(node, 0) == 0 and node != source_node and node != sink_node:
            edges.append((node, sink_node))

    return nodes, edges
```

- **Purpose**: Orchestrates the parsing and graph-building process.
- **Process**:
  1. **Tokenization**: Converts the input string into tokens.
  2. **Initialization**: Resets global variables.
  3. **Parsing**: Calls `parse_expr` to parse the tokens and build the initial nodes and edges.
  4. **Adding Source and Sink Nodes**:
     - Creates special nodes named 'source' and 'sink'.
     - Adds them to the `nodes` list.
  5. **Determining Node Connectivity**:
     - Creates dictionaries `node_incoming` and `node_outgoing` to count the number of incoming and outgoing edges for each node.
  6. **Connecting Source and Sink**:
     - Adds edges from the 'source' node to nodes with no incoming edges.
     - Adds edges from nodes with no outgoing edges to the 'sink' node.
  7. **Returns**: The updated `nodes` and `edges` lists.

---

#### Output:
```
Nodes:
- 1_K01657
- 2_K01658
- 3_K13503
- 4_K13501
- 5_K01656
- 6_K00766
- 7_K13497
- 8_K01817
- 9_K24017
- 10_K01656
- 11_K01609
- 12_K13498
- 13_K13501
- 14_K01695
- 15_K01696
- 16_K06001
- 17_K01694
- 18_source
- 19_sink
```
```
Edges:
1_K01657 -> 2_K01658
2_K01658 -> 6_K00766
3_K13503 -> 6_K00766
4_K13501 -> 6_K00766
5_K01656 -> 6_K00766
8_K01817 -> 10_K01656
8_K01817 -> 11_K01609
9_K24017 -> 10_K01656
9_K24017 -> 11_K01609
6_K00766 -> 8_K01817
6_K00766 -> 9_K24017
6_K00766 -> 12_K13498
6_K00766 -> 13_K13501
7_K13497 -> 8_K01817
7_K13497 -> 9_K24017
7_K13497 -> 12_K13498
7_K13497 -> 13_K13501
14_K01695 -> 15_K01696
14_K01695 -> 16_K06001
10_K01656 -> 14_K01695
10_K01656 -> 17_K01694
11_K01609 -> 14_K01695
11_K01609 -> 17_K01694
12_K13498 -> 14_K01695
12_K13498 -> 17_K01694
13_K13501 -> 14_K01695
13_K13501 -> 17_K01694
18_source -> 1_K01657
18_source -> 3_K13503
18_source -> 4_K13501
18_source -> 5_K01656
18_source -> 7_K13497
15_K01696 -> 19_sink
16_K06001 -> 19_sink
17_K01694 -> 19_sink
```

### Vizualization

Our vizualization function uses the graphviz dot layout to arrange nodes hierarchically, which is ideal for representing flows from a source to a sink in a directed graph. It places nodes in layers, respecting the direction of edges.

```python
def visualize_graph(nodes, edges):
    G = nx.DiGraph()
    for node in nodes:
        G.add_node(str(node))
    for edge in edges:
        if len(edge) == 3:
            from_node, to_node, label = edge
            G.add_edge(str(from_node), str(to_node), label=label)
        else:
            from_node, to_node = edge
            G.add_edge(str(from_node), str(to_node))
    
    # Use Graphviz's 'dot' layout
    pos = nx.nx_pydot.graphviz_layout(G, prog='dot')
    
    plt.figure(figsize=(12, 8))
    nx.draw(G, pos, with_labels=True, node_size=2000, node_color='lightblue', arrows=True, arrowstyle='->', arrowsize=20)
    
    # Draw edge labels if any
    edge_labels = nx.get_edge_attributes(G, 'label')
    if edge_labels:
        nx.draw_networkx_edge_labels(G, pos, edge_labels=edge_labels)
    
    plt.title("Directed Graph of Module M00023")
    plt.axis('off')
    plt.show()
```

#### Output

![_posts/Pictures/1_KEGGReconstructed.png](https://github.com/NehaSontakk/github-pages/blob/main/_posts/Pictures/1_KEGGReconstructed.png)

### Comparison with the KEGG Website

KEGG contains hand drawn images of its modules, and their representation for M00023 is given below:

![_posts/Pictures/2_ko_M00023.png](https://github.com/NehaSontakk/github-pages/blob/main/_posts/Pictures/2_ko_M00023.png)
