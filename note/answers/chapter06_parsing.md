1.  The comma operator has the lowest precedence, so it goes between expression
    and equality:

    ```lox
    expression → comma
    comma      → equality ( "," equality )*
    equality   → comparison ( ( "!=" | "==" ) comparison )*
    comparison → term ( ( ">" | ">=" | "<" | "<=" ) term )*
    term       → factor ( ( "-" | "+" ) factor )*
    factor     → unary ( ( "/" | "*" ) unary )*
    unary      → ( "!" | "-" | "--" | "++" ) unary
               | postfix
    postfix    → primary ( "--" | ++" )*
    primary    → NUMBER | STRING | "true" | "false" | "nil"
               | "(" expression ")"
    ```

    We could define a new syntax tree node by adding this to the `defineAst()`
    call:

    ```java
    "Comma    : Expr left, Expr right",
    ```

    But a simpler choice is to treat it like any other binary operator and
    reuse Expr.Binary.

    Parsing is similar to other infix operators (except that we don't bother to
    keep the operator token):

    ```java
    private Expr expression() {
      return comma();
    }

    private Expr comma() {
      Expr expr = equality();

      while (match(COMMA)) {
        Token operator = previous();
        Expr right = equality();
        expr = new Expr.Binary(expr, operator, right);
      }

      return expr;
    }
    ```

2.  We just need one new rule.

    ```lox
    expression  → conditional
    conditional → equality ( "?" expression ":" conditional )?
    // Other rules...
    ```

    The precedence of the operands is pretty interesting. The left operand has
    higher precedence than the others, and the middle operand has lower
    precedence than the condition expression itself. That allows:

        a ? b = c : d

    Again, I won't bother showing the scanner and token changes since they're
    pretty obvious.

    private Expr expression() {
      return conditional();
    }

    private Expr conditional() {
      Expr expr = equality();

      if (match(QUESTION)) {
        Expr thenBranch = expression();
        consume(":", "Expect ':' after then branch of conditional expression.");
        Expr elseBranch = conditional();
        expr = new Expr.Conditional(expr, thenBranch, elseBranch);
      }

      return expr;
    }

3.  Here's an updated grammar. The grammar itself doesn't "know" that some of
    these productions are errors. The parser handles that.

    ```lox
    expression → equality
    equality   → comparison ( ( "!=" | "==" ) comparison )*
    comparison → term ( ( ">" | ">=" | "<" | "<=" ) term )*
    term       → factor ( ( "-" | "+" ) factor )*
    factor     → unary ( ( "/" | "*" ) unary )*
    unary      → ( "!" | "-" | "--" | "++" ) unary
               | postfix
    postfix    → primary ( "--" | ++" )*
    primary    → NUMBER | STRING | "true" | "false" | "nil"
               | "(" expression ")"
               // Error productions...
               | ( "!=" | "==" ) equality
               | ( ">" | ">=" | "<" | "<=" ) comparison
               | ( "+" ) term
               | ( "/" | "*" ) factor
    ```

    Things to note:

    * "-" isn't an error production because that *is* a valid prefix
      expression.

    * The precedence for each operator is one level higher than it is for the
      normal correct. We also don't parse a sequence, just a single RHS. Using
      the same precedence level handles a sequence for us and helps us only
      show the error once?

    private Expr primary() {
      if (match(FALSE)) return new Expr.Literal(false);
      if (match(TRUE)) return new Expr.Literal(true);
      if (match(NIL)) return new Expr.Literal(null);

      if (match(NUMBER, STRING)) {
        return new Expr.Literal(previous().literal);
      }

      if (match(LEFT_PAREN)) {
        Expr expr = expression();
        consume(RIGHT_PAREN, "Expect ')' after expression.");
        return new Expr.Grouping(expr);
      }

      // Error productions.
      if (match(BANG_EQUAL, EQUAL_EQUAL)) {
        error(previous(), "Missing left-hand operand.");
        equality();
        return null;
      }

      if (match(GREATER, GREATER_EQUAL, LESS, LESS_EQUAL)) {
        error(previous(), "Missing left-hand operand.");
        comparison();
        return null;
      }

      if (match(PLUS)) {
        error(previous(), "Missing left-hand operand.");
        term();
        return null;
      }

      if (match(SLASH, STAR)) {
        error(previous(), "Missing left-hand operand.");
        factor();
        return null;
      }

      throw error(peek(), "Expect expression.");
    }