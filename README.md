## CoreNLP wrapper for Apache Spark

### CoreNLP annotators as DataFrame functions in Spark SQL

Following the [simple APIs](http://stanfordnlp.github.io/CoreNLP/simple.html) introduced in Stanford
CoreNLP 3.6.0, we implemented DataFrame functions to wrap those APIs, which are simple to use but
less customizable.
All functions are defined under `com.databricks.spark.corenlp.functions`.

* *`cleanxml`*: Cleans XML tags in a document and returns the cleaned document.
* *`tokenize`*: Tokenizes a sentence into words.
* *`ssplit`*: Splits a document into sentences.
* *`pos`*: Generates the part of speech tags of the sentence.
* *`lemma`*: Generates the word lemmas of the sentence.
* *`ner`*: Generates the named entity tags of the sentence.
* *`depparse`*: Generates the semantic dependencies of the sentence and returns a flattened list of
  `(source, sourceIndex, relation, target, targetIndex, weight)` relation tuples.
* *`coref`*: Generates the coref chains in the document and returns a list of
  `(rep, mentions)` chain tuples, where `mentions` are in the format of
  `(sentNum, startIndex, mention)`.
* *`natlog`*: Generates the Natural Logic notion of polarity for each token in a sentence, returned
  as `up`, `down`, or `flat`.
* *`openie`*: Generates a list of Open IE triples as flat `(subject, relation, target, confidence)`
  tuples.
* *`sentiment`*: Measures the sentiment of an input sentence on a scale of 0 (strong negative) to 4
  (strong positive).  

Users can chain the functions to create pipeline, for example:

~~~scala
import org.apache.spark.sql.functions._
import com.databricks.spark.corenlp.functions._

import sqlContext.implicits._

val input = Seq(
  (1, "<xml>Stanford University is located in California. It is a great university.</xml>")
).toDF("id", "text")

val output = input
  .select(cleanxml('text).as('doc))
  .select(explode(ssplit('doc)).as('sen))
  .select('sen, tokenize('sen).as('words), ner('sen).as('nerTags), sentiment('sen).as('sentiment))

output.show(truncate = false)
~~~

~~~
+----------------------------------------------+------------------------------------------------------+--------------------------------------------------+---------+
|sen                                           |words                                                 |nerTags                                           |sentiment|
+----------------------------------------------+------------------------------------------------------+--------------------------------------------------+---------+
|Stanford University is located in California .|[Stanford, University, is, located, in, California, .]|[ORGANIZATION, ORGANIZATION, O, O, O, LOCATION, O]|1        |
|It is a great university .                    |[It, is, a, great, university, .]                     |[O, O, O, O, O, O]                                |4        |
+----------------------------------------------+------------------------------------------------------+--------------------------------------------------+---------+
~~~

### CoreNLP as a Transformer in Spark ML pipelines API

`com.databricks.spark.corenlp.CoreNLP` wraps
[Stanford CoreNLP](http://nlp.stanford.edu/software/corenlp.shtml) annotation pipeline as an
`org.apache.spark.ml.Transformer`.
It reads a string column representing documents, and applies CoreNLP annotators to each document.
The output column is a nested column with schema mapped from the CoreNLP Document proto:

~~~
root
 |-- text: string (nullable = true)
 |-- sentence: array (nullable = true)
 |    |-- element: struct (containsNull = true)
 |    |    |-- token: array (nullable = true)
 |    |    |    |-- element: struct (containsNull = true)
 |    |    |    |    |-- word: string (nullable = true)
 |    |    |    |    |-- pos: string (nullable = true)
 |    |    |    |    |-- value: string (nullable = true)
 |    |    |    |    |-- category: string (nullable = true)
 |    |    |    |    |-- before: string (nullable = true)
 |    |    |    |    |-- after: string (nullable = true)
 |    |    |    |    |-- originalText: string (nullable = true)
 |    |    |    |    |-- ner: string (nullable = true)
 |    |    |    |    |-- normalizedNER: string (nullable = true)
 |    |    |    |    |-- lemma: string (nullable = true)
 |    |    |    |    |-- beginChar: integer (nullable = true)
 |    |    |    |    |-- endChar: integer (nullable = true)
 |    |    |    |    |-- utterance: integer (nullable = true)
 |    |    |    |    |-- speaker: string (nullable = true)
 |    |    |    |    |-- beginIndex: integer (nullable = true)
 |    |    |    |    |-- endIndex: integer (nullable = true)
 |    |    |    |    |-- tokenBeginIndex: integer (nullable = true)
 |    |    |    |    |-- tokenEndIndex: integer (nullable = true)
 |    |    |    |    |-- timexValue: null (nullable = true)
 |    |    |    |    |-- hasXmlContext: boolean (nullable = true)
 |    |    |    |    |-- xmlContext: array (nullable = true)
 |    |    |    |    |    |-- element: string (containsNull = true)
 |    |    |    |    |-- corefClusterID: integer (nullable = true)
 |    |    |    |    |-- answer: string (nullable = true)
 |    |    |    |    |-- headWordIndex: integer (nullable = true)
 |    |    |    |    |-- operator: null (nullable = true)
 |    |    |    |    |-- polarity: null (nullable = true)
 |    |    |    |    |-- span: null (nullable = true)
 |    |    |    |    |-- sentiment: string (nullable = true)
 |    |    |    |    |-- quotationIndex: integer (nullable = true)
 |    |    |    |    |-- gender: string (nullable = true)
 |    |    |    |    |-- trueCase: string (nullable = true)
 |    |    |    |    |-- trueCaseText: string (nullable = true)
 |    |    |-- tokenOffsetBegin: integer (nullable = true)
 |    |    |-- tokenOffsetEnd: integer (nullable = true)
 |    |    |-- sentenceIndex: integer (nullable = true)
 |    |    |-- characterOffsetBegin: integer (nullable = true)
 |    |    |-- characterOffsetEnd: integer (nullable = true)
 |    |    |-- parseTree: struct (nullable = true)
 |    |    |    |-- child: array (nullable = true)
 |    |    |    |    |-- element: null (containsNull = true)
 |    |    |    |-- value: string (nullable = true)
 |    |    |    |-- yieldBeginIndex: integer (nullable = true)
 |    |    |    |-- yieldEndIndex: integer (nullable = true)
 |    |    |    |-- score: double (nullable = true)
 |    |    |    |-- sentiment: string (nullable = true)
 |    |    |-- binarizedParseTree: struct (nullable = true)
 |    |    |    |-- child: array (nullable = true)
 |    |    |    |    |-- element: null (containsNull = true)
 |    |    |    |-- value: string (nullable = true)
 |    |    |    |-- yieldBeginIndex: integer (nullable = true)
 |    |    |    |-- yieldEndIndex: integer (nullable = true)
 |    |    |    |-- score: double (nullable = true)
 |    |    |    |-- sentiment: string (nullable = true)
 |    |    |-- annotatedParseTree: struct (nullable = true)
 |    |    |    |-- child: array (nullable = true)
 |    |    |    |    |-- element: null (containsNull = true)
 |    |    |    |-- value: string (nullable = true)
 |    |    |    |-- yieldBeginIndex: integer (nullable = true)
 |    |    |    |-- yieldEndIndex: integer (nullable = true)
 |    |    |    |-- score: double (nullable = true)
 |    |    |    |-- sentiment: string (nullable = true)
 |    |    |-- sentiment: string (nullable = true)
 |    |    |-- basicDependencies: struct (nullable = true)
 |    |    |    |-- node: array (nullable = true)
 |    |    |    |    |-- element: null (containsNull = true)
 |    |    |    |-- edge: array (nullable = true)
 |    |    |    |    |-- element: null (containsNull = true)
 |    |    |    |-- root: array (nullable = true)
 |    |    |    |    |-- element: integer (containsNull = true)
 |    |    |-- collapsedDependencies: struct (nullable = true)
 |    |    |    |-- node: array (nullable = true)
 |    |    |    |    |-- element: null (containsNull = true)
 |    |    |    |-- edge: array (nullable = true)
 |    |    |    |    |-- element: null (containsNull = true)
 |    |    |    |-- root: array (nullable = true)
 |    |    |    |    |-- element: integer (containsNull = true)
 |    |    |-- collapsedCCProcessedDependencies: struct (nullable = true)
 |    |    |    |-- node: array (nullable = true)
 |    |    |    |    |-- element: null (containsNull = true)
 |    |    |    |-- edge: array (nullable = true)
 |    |    |    |    |-- element: null (containsNull = true)
 |    |    |    |-- root: array (nullable = true)
 |    |    |    |    |-- element: integer (containsNull = true)
 |    |    |-- alternativeDependencies: struct (nullable = true)
 |    |    |    |-- node: array (nullable = true)
 |    |    |    |    |-- element: null (containsNull = true)
 |    |    |    |-- edge: array (nullable = true)
 |    |    |    |    |-- element: null (containsNull = true)
 |    |    |    |-- root: array (nullable = true)
 |    |    |    |    |-- element: integer (containsNull = true)
 |    |    |-- openieTriple: array (nullable = true)
 |    |    |    |-- element: struct (containsNull = true)
 |    |    |    |    |-- subject: string (nullable = true)
 |    |    |    |    |-- relation: string (nullable = true)
 |    |    |    |    |-- object: string (nullable = true)
 |    |    |    |    |-- confidence: double (nullable = true)
 |    |    |    |    |-- subjectSpan: null (nullable = true)
 |    |    |    |    |-- relationSpan: null (nullable = true)
 |    |    |    |    |-- objectSpan: null (nullable = true)
 |    |    |    |    |-- tree: null (nullable = true)
 |    |    |-- paragraph: integer (nullable = true)
 |    |    |-- text: string (nullable = true)
 |    |    |-- hasRelationAnnotations: boolean (nullable = true)
 |    |    |-- entity: array (nullable = true)
 |    |    |    |-- element: struct (containsNull = true)
 |    |    |    |    |-- headStart: integer (nullable = true)
 |    |    |    |    |-- headEnd: integer (nullable = true)
 |    |    |    |    |-- mentionType: string (nullable = true)
 |    |    |    |    |-- normalizedName: string (nullable = true)
 |    |    |    |    |-- headTokenIndex: integer (nullable = true)
 |    |    |    |    |-- corefID: string (nullable = true)
 |    |    |    |    |-- objectID: string (nullable = true)
 |    |    |    |    |-- extentStart: integer (nullable = true)
 |    |    |    |    |-- extentEnd: integer (nullable = true)
 |    |    |    |    |-- type: string (nullable = true)
 |    |    |    |    |-- subtype: string (nullable = true)
 |    |    |-- relation: array (nullable = true)
 |    |    |    |-- element: struct (containsNull = true)
 |    |    |    |    |-- argName: array (nullable = true)
 |    |    |    |    |    |-- element: string (containsNull = true)
 |    |    |    |    |-- arg: array (nullable = true)
 |    |    |    |    |    |-- element: null (containsNull = true)
 |    |    |    |    |-- signature: string (nullable = true)
 |    |    |    |    |-- objectID: string (nullable = true)
 |    |    |    |    |-- extentStart: integer (nullable = true)
 |    |    |    |    |-- extentEnd: integer (nullable = true)
 |    |    |    |    |-- type: string (nullable = true)
 |    |    |    |    |-- subtype: string (nullable = true)
 |    |    |-- hasNumerizedTokensAnnotation: boolean (nullable = true)
 |    |    |-- mentions: array (nullable = true)
 |    |    |    |-- element: struct (containsNull = true)
 |    |    |    |    |-- sentenceIndex: integer (nullable = true)
 |    |    |    |    |-- tokenStartInSentenceInclusive: integer (nullable = true)
 |    |    |    |    |-- tokenEndInSentenceExclusive: integer (nullable = true)
 |    |    |    |    |-- ner: string (nullable = true)
 |    |    |    |    |-- normalizedNER: string (nullable = true)
 |    |    |    |    |-- entityType: string (nullable = true)
 |    |    |    |    |-- timex: null (nullable = true)
 |-- corefChain: array (nullable = true)
 |    |-- element: struct (containsNull = true)
 |    |    |-- chainID: integer (nullable = true)
 |    |    |-- mention: array (nullable = true)
 |    |    |    |-- element: struct (containsNull = true)
 |    |    |    |    |-- mentionID: integer (nullable = true)
 |    |    |    |    |-- mentionType: string (nullable = true)
 |    |    |    |    |-- number: string (nullable = true)
 |    |    |    |    |-- gender: string (nullable = true)
 |    |    |    |    |-- animacy: string (nullable = true)
 |    |    |    |    |-- beginIndex: integer (nullable = true)
 |    |    |    |    |-- endIndex: integer (nullable = true)
 |    |    |    |    |-- headIndex: integer (nullable = true)
 |    |    |    |    |-- sentenceIndex: integer (nullable = true)
 |    |    |    |    |-- position: integer (nullable = true)
 |    |    |-- representative: integer (nullable = true)
 |-- docID: string (nullable = true)
 |-- sentencelessToken: array (nullable = true)
 |    |-- element: struct (containsNull = true)
 |    |    |-- word: string (nullable = true)
 |    |    |-- pos: string (nullable = true)
 |    |    |-- value: string (nullable = true)
 |    |    |-- category: string (nullable = true)
 |    |    |-- before: string (nullable = true)
 |    |    |-- after: string (nullable = true)
 |    |    |-- originalText: string (nullable = true)
 |    |    |-- ner: string (nullable = true)
 |    |    |-- normalizedNER: string (nullable = true)
 |    |    |-- lemma: string (nullable = true)
 |    |    |-- beginChar: integer (nullable = true)
 |    |    |-- endChar: integer (nullable = true)
 |    |    |-- utterance: integer (nullable = true)
 |    |    |-- speaker: string (nullable = true)
 |    |    |-- beginIndex: integer (nullable = true)
 |    |    |-- endIndex: integer (nullable = true)
 |    |    |-- tokenBeginIndex: integer (nullable = true)
 |    |    |-- tokenEndIndex: integer (nullable = true)
 |    |    |-- timexValue: struct (nullable = true)
 |    |    |    |-- value: string (nullable = true)
 |    |    |    |-- altValue: string (nullable = true)
 |    |    |    |-- text: string (nullable = true)
 |    |    |    |-- type: string (nullable = true)
 |    |    |    |-- tid: string (nullable = true)
 |    |    |    |-- beginPoint: integer (nullable = true)
 |    |    |    |-- endPoint: integer (nullable = true)
 |    |    |-- hasXmlContext: boolean (nullable = true)
 |    |    |-- xmlContext: array (nullable = true)
 |    |    |    |-- element: string (containsNull = true)
 |    |    |-- corefClusterID: integer (nullable = true)
 |    |    |-- answer: string (nullable = true)
 |    |    |-- headWordIndex: integer (nullable = true)
 |    |    |-- operator: struct (nullable = true)
 |    |    |    |-- name: string (nullable = true)
 |    |    |    |-- quantifierSpanBegin: integer (nullable = true)
 |    |    |    |-- quantifierSpanEnd: integer (nullable = true)
 |    |    |    |-- subjectSpanBegin: integer (nullable = true)
 |    |    |    |-- subjectSpanEnd: integer (nullable = true)
 |    |    |    |-- objectSpanBegin: integer (nullable = true)
 |    |    |    |-- objectSpanEnd: integer (nullable = true)
 |    |    |-- polarity: struct (nullable = true)
 |    |    |    |-- projectEquivalence: string (nullable = true)
 |    |    |    |-- projectForwardEntailment: string (nullable = true)
 |    |    |    |-- projectReverseEntailment: string (nullable = true)
 |    |    |    |-- projectNegation: string (nullable = true)
 |    |    |    |-- projectAlternation: string (nullable = true)
 |    |    |    |-- projectCover: string (nullable = true)
 |    |    |    |-- projectIndependence: string (nullable = true)
 |    |    |-- span: struct (nullable = true)
 |    |    |    |-- begin: integer (nullable = true)
 |    |    |    |-- end: integer (nullable = true)
 |    |    |-- sentiment: string (nullable = true)
 |    |    |-- quotationIndex: integer (nullable = true)
 |    |    |-- gender: string (nullable = true)
 |    |    |-- trueCase: string (nullable = true)
 |    |    |-- trueCaseText: string (nullable = true)
 |-- quote: array (nullable = true)
 |    |-- element: struct (containsNull = true)
 |    |    |-- text: string (nullable = true)
 |    |    |-- begin: integer (nullable = true)
 |    |    |-- end: integer (nullable = true)
 |    |    |-- sentenceBegin: integer (nullable = true)
 |    |    |-- sentenceEnd: integer (nullable = true)
 |    |    |-- tokenBegin: integer (nullable = true)
 |    |    |-- tokenEnd: integer (nullable = true)
 |    |    |-- docid: string (nullable = true)
 |    |    |-- index: integer (nullable = true)
~~~

Further pruning and filtering could be done via SQL statements, or via `flattenNestedFields` param.
For example,

~~~scala
val input = sqlContext.createDataFrame(Seq(
  (1, "<xml>Stanford University is located in California. It is a great university.</xml>")
)).toDF("id", "text")
val coreNLP = new CoreNLP()
  .setInputCol("text")
  .setAnnotators(Array("tokenize", "cleanxml", "ssplit"))
  .setFlattenNestedFields(Array("sentence_token_word", "sentence_characterOffsetBegin"))
  .setOutputCol("parsed")
val parsed = coreNLP.transform(input)
  .select("parsed.sentence_token_word", "parsed.sentence_characterOffsetBegin")
println(parsed.first())
~~~

produces the following output:

~~~
[ArrayBuffer(Stanford, University, is, located, in, California, ., It, is, a, great, university, .),ArrayBuffer(5, 51)]
~~~

This package requires Java 8 and CoreNLP 3.6.0 to run.
Users must include CoreNLP model jars as dependencies to use language models.
