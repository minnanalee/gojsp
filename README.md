## Very early attempt at creating an [ANTLR](https://www.antlr.org/) visitor for javascript files using GO. Going to change like Irish weather so don't expect things to stay the same any time soon. 


The `base` is the result of running what is described [here](https://github.com/padraicbc/gojsp/tree/master/runantlr#readme)

The [vast](https://github.com/padraicbc/gojsp/tree/master/vast) folder is the implementation so far toward creating an ast which will allow easy manipulation/translation of the original source.
Thre are a few example functions in the exampls folder including how to parse and [manipulate](https://github.com/padraicbc/gojsp/blob/master/examples/singleexpress.go) a single expression. 


    import (
	"log"

	antlr "github.com/padraicbc/antlr4"
	"github.com/padraicbc/gojsp/base"
	"github.com/padraicbc/gojsp/vast"
    )

stream := antlr.NewInputStream(`i + j;`)
	lexer := base.NewJavaScriptLexer(stream)

	tokenStream := antlr.NewCommonTokenStream(lexer, antlr.TokenDefaultChannel)

	p := base.NewJavaScriptParser(tokenStream)
	// start at ExpressionSequence
	tree := p.ExpressionSequence()
	v := vast.NewVisitor(lexer.SymbolicNames, p.GetRuleNames())
	exp := visit(tree, v).(*vast.ExpressionSequence)

	expc := exp.FirstChild().(*vast.LRExpression)

	fmt.Println(expc.Left().(vast.Token).Value(), expc.OP().Value(), expc.Right().(vast.Token).Value())
	// change OP
	expc.OP().SetValue("/")
	expc.Right().(vast.Token).SetValue("1000")
	fmt.Println(expc.Left().(vast.Token).Value(), expc.OP().Value(), expc.Right().(vast.Token).Value())

	// reuse lexer and parser
	stream.Seek(0)
	lexer.SetInputStream(stream)
	tokenStream = antlr.NewCommonTokenStream(lexer, antlr.TokenDefaultChannel)
	p.SetInputStream(tokenStream)

	v = vast.NewVisitor(lexer.SymbolicNames, p.GetRuleNames())
	// alternative using Body
	tree2 := p.Program()
	v = vast.NewVisitor(lexer.SymbolicNames, p.GetRuleNames())
	exp2 := visit(tree2, v).(*vast.Program).Body[0].(*vast.ExpressionStatement).FirstChild()

	expc = exp2.FirstChild().(*vast.LRExpression)
	// can be any singleExpression so any VNode
	fmt.Println(expc.Left().(vast.Token).Value(), expc.OP().Value(), expc.Right().(vast.Token).Value())

And a very incomplete conversion from arrow to es5 functions but it shows the general idea:

		stream := antlr.NewInputStream(`

    // Arrow Function Break Down

    // 1. Remove the word "function" and place arrow between the argument and opening body bracket
    (a) => {
    return a + 100;
    }

    // 2. Remove the body brackets and word "return" -- the return is implied.
    (b) => b + 100;

    // 3. Remove the argument parentheses
    c => c 

    // Arrow Function
    (a, b) => {
    let chuck = 42;
    return a + b + chuck;
    }
    `)
	lexer := base.NewJavaScriptLexer(stream)

	tokenStream := antlr.NewCommonTokenStream(lexer, antlr.TokenDefaultChannel)

	p := base.NewJavaScriptParser(tokenStream)

	tree := p.Program()

	v := vast.NewVisitor(lexer.SymbolicNames, p.GetRuleNames())
	// v.Debug = true

	rfs := visit(tree, v).(*vast.Program).Body
	// *ExpressionStatememts -> ExpressionSequence, iterate and check types
	for _, fn := range rfs {
		log.Println(fn.Type())
		var trans string
		// all with one child
		af := fn.FirstChild().FirstChild().(*vast.ArrowFunction)
		fmt.Println("Before ->", af.Code())
		// either has a fucntion body with {} of a single expression.
		if af.Body.FirstChild() != nil {
			// can be there or not
			var open, close string
			if af.FunctionParameters.OpenParen == nil {
				open, close = "(", ")"
			}
			log.Println(af.Body.FirstChild().Type(), af.FunctionParameters.Source)
			trans = fmt.Sprintf("function%s%s%s {\n\treturn %s\n}",
				open, af.FunctionParameters.Source, close, af.Body.FirstChild().GetInfo().Source)

		}
		if af.Body != nil {

			trans = fmt.Sprintf("function%s %s", af.FunctionParameters.Source, af.Body.Source)

		}
		fmt.Println("After ->", trans)

	}
	
Quite verbose but more about getting it working than pretty to start...
