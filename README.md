# semantic-snatcher

a deno project that will extract triples and a object URI from a given source URI

### Step-by-Step Plan for the semantic snatching

#### 1. **Initialize Function**

- Input: A URI that will be used to retrieve triples.
- Output: A triplestore and the subject URI that describes the object (if found).

#### 2. **Retrieve Initial Triples**

- **2.1 Send HTTP GET Request with Accept Headers:**
  - **First Accept Header:** `text/turtle`
  - **Second Accept Header:** `application/ld+json`
  - **Third Accept Header:** `text/html`
- **2.2** For each response, check:
  - **If `text/turtle` or `application/ld+json` is successful** (contains triples):
    - Parse the content and add it to the triplestore.
    - **Exit this step** if triples are found.
  - **If `text/html` is used**:
    - Parse for embedded `application/ld+json` or `text/turtle` in the HTML.
    - **Look for JSON-LD content in `<script>` tags**:
      - `<script>` tag with `type="application/ld+json"`.
      - **Or** `<link>` tag with `rel="describedby"` and `type="application/ld+json"`.
    - If found, **fetch content**, parse JSON-LD, and add it to the triplestore.
  - **If no triples are found in all Accept headers**, exit the function without a triplestore.

#### 3. **Check if Triples Are Found**

- If the triplestore is empty after Step 2, **terminate** the function.
- If triples are found, proceed to find the `subject URI` within the triplestore.

#### 4. **Query Triplestore for Subject URI**

- **Primary Query:** Use the original URI as the subject in a query.
- If triples are returned from this query:
  - **Check** if any triple has the predicate `schema:describes`.
  - If found:
    - **Use the object of this triple** as the new subject URI.
    - **Exit** this step.
- **Secondary Query (Redirected URI Check):**
  - If the URI experienced redirects during Step 2, use the **final redirected URI** as the subject in a query.
  - Repeat the check for `schema:describes` predicate.
  - If found, **use this URI as the subject URI** and **exit** this step.

#### 5. **Fallback Query with URI as Object**

- If no triples were found in Step 4, **use the URI as the object** in a query.
- If triples are found, use the **subject of this query as the subject URI**.
- **Check the predicate** used to find the subject in this query for additional logic if needed.

#### 6. **Return Results**

- Return the populated `triplestore` and the `subject URI` that describes the object.

```mermaid
flowchart TD
    A[Start: Initialize with URI] --> B[Send GET Request with Accept Headers]
    B --> C{Response Content}
    C -->|text/turtle or application/ld+json| D[Parse and Store Triples]
    C -->|text/html| E[Parse HTML for Embedded Triples]
    E --> F{Embedded JSON-LD or Turtle Found?}
    F -->|Yes| D
    F -->|No| G[Check Next Accept Header]
    G --> B

    D --> H{Triplestore Populated?}
    H -->|No| I[Return No Triples Found]
    H -->|Yes| J[Query Triplestore for Subject URI]

    J --> K{Triples Found for Original URI?}
    K -->|Yes| L[Check for schema:describes Predicate]
    L -->|Yes| M[Return Triplestore and Object as Subject URI]
    L -->|No| N[Use Original URI as Describing Subject URI]
    K -->|No| O{Was URI Redirected?}

    O -->|Yes| P[Query with Final Redirected URI]
    P --> Q{Triples Found?}
    Q -->|Yes| R[Check for schema:describes Predicate]
    R -->|Yes| M
    R -->|No| S[Query Triplestore with Original URI as Object]

    Q -->|No| S
    O -->|No| S
    S --> T{Triples Found for URI as Object?}
    T -->|Yes| U[Use Subject of Triple as Subject URI]
    U --> M
    T -->|No| I

```

---

### Sequence Diagram Description

1. **User/System** invokes the function with an initial URI.
2. **Function** sends HTTP GET requests with specified Accept headers (`text/turtle`, `application/ld+json`, and `text/html`).
   - For each response, it tries to parse and store triples in the triplestore.
   - If `text/html` is used, **HTML Parser** looks for embedded JSON-LD or Turtle content.
3. **Function checks the triplestore** to see if triples were found:
   - If empty, **returns** without data.
   - If populated, proceeds to subject URI search.
4. **Function performs query** with the original URI as the subject:
   - If successful, checks for `schema:describes`.
   - If found, uses the triple’s object as the `subject URI` and **returns the triplestore and subject URI**.
5. If no relevant triples are found and if there were redirects:
   - **Function performs query with the final redirected URI** as the subject.
6. If still no results, **Function performs query using the URI as an object**.
7. If triples are returned, uses the subject of the triple as the `subject URI`.
8. **Return Result**:
   - Function returns the `triplestore` and the determined `subject URI`.

```mermaid
sequenceDiagram
    participant User
    participant Function
    participant HTTPService as HTTP Service
    participant HTMLParser as HTML Parser
    participant Triplestore

    User->>Function: Call function with initial URI
    loop Accept Headers
        Function->>HTTPService: Send GET request with Accept headers (text/turtle, application/ld+json, text/html)
        alt Triple response found
            HTTPService-->>Function: Return triples in turtle or JSON-LD
            Function->>Triplestore: Parse and store triples
        else HTML Content
            HTTPService-->>Function: Return HTML content
            Function->>HTMLParser: Parse HTML for embedded JSON-LD or Turtle
            alt Embedded JSON-LD found
                HTMLParser-->>Function: Return JSON-LD
                Function->>Triplestore: Parse and store triples
            else No embedded content found
                HTMLParser-->>Function: No triples found
            end
        else No triples found
            HTTPService-->>Function: No content found
        end
    end

    alt Triplestore is empty
        Function-->>User: Return empty result (No triples found)
    else Triples found in Triplestore
        Function->>Triplestore: Query for subject URI using original URI
        alt Triples found with schema:describes predicate
            Triplestore-->>Function: Return object as subject URI
            Function-->>User: Return triplestore and subject URI
        else No triples found
            alt Final redirected URI available
                Function->>Triplestore: Query using redirected URI as subject
                alt schema:describes found
                    Triplestore-->>Function: Return object as subject URI
                    Function-->>User: Return triplestore and subject URI
                else No triples found
                    Function->>Triplestore: Query using URI as object
                    alt Triples found
                        Triplestore-->>Function: Return subject as subject URI
                        Function-->>User: Return triplestore and subject URI
                    else No triples found
                        Function-->>User: Return triplestore (No describing subject found)
                    end
                end
            else Query URI as object
                Function->>Triplestore: Query using URI as object
                alt Triples found
                    Triplestore-->>Function: Return subject as subject URI
                    Function-->>User: Return triplestore and subject URI
                else No triples found
                    Function-->>User: Return triplestore (No describing subject found)
                end
            end
        end
    end
```
