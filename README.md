hadoop-hocr-parser
==================

Background
----------

This project contains a Hadoop job for parsing a large collection of HTML book 
page layout and text content ﬁles (more precisely, hOCR [1] ﬁles) in order to 
extract structured information that can be used to identify possible quality 
issues. The hOCR HTML ﬁles are part of a book collection where each HTML page 
represents layout and text of a corresponding book page image. These HTML
ﬁles have block level elements described by properties of a ’div’ HTML tag. 
Each ’div’ element has a ”position”, ”width” and ”height” property representing 
the surrounding rectangle of a text or image block.

Assuming a typical book page layout, the identiﬁed text block width (multiple 
columns summing up to one text block) plus the white margins should approximately 
match the book page image width. Based on this assumption, the hypothesis 
is that if there is a signiﬁcant mismatch for several pages of the same book, 
it can be interpreted as an indicator for possible cropping errors with text loss.

This project shows the principle of approaching this kind of data processing 
scenario by means of the MapReduce programming model. The HTML parsing is done
using JSoup [2] and is executed in parallel in the Map phase. Subsequently, the 
average text block width per page is calculated in the Reduce phase.

Data preparation
----------------

First of all, dealing with lots of HTML files, means that we are facing 
Hadoop’s “Small Files Problem” [3]. In brief, this is to say that the files we 
want to process are too small for taking them directly as input for the Map 
function. 

One approach to overcome this is to put all the small files into one large 
SequenceFile which is an input format that can be used to aggregate content 
using some identifier as key and the byte sequence of the content as value. 

The input for this hadoop job is therefore a SequenceFile aggregating the
individual HTML files in one large file, see the related project:

https://github.com/shsdev/sequencefile-utility

Hadoop MapReduce job
--------------------

Once data is loaded into HDFS, the SequenceFileInputFormat can be used as input 
in the subsequent MapReduce job which parses the HTML ﬁles using the Java HTML 
parser Jsoup in the Map function and calculates the average block width in the 
Reduce function.

Let k1 be the identiﬁer of the HTML ﬁle (data type: org.apache.hadoop.io.Text), 
and v1 be the value holding the content of the HTML ﬁle (data type: 
org.apache.hadoop.io.BytesWritable). The key-value pair (k1;v1) is the input of 
the Map function shown in the following equation.

    map(k1, v1) --> list(k1, v2)

As a simple example, let us assume one key-value pair input with the page 
identiﬁer 00001.html and the HTML ﬁle content as value as shown in the following 
example:

    ("00001.html", "<html>[content]</html>")

The mapper will produce a list of key-value pairs, one key-value pair for each 
text block the parser ﬁnds, as output. Assuming the parser ﬁnds two text blocks 
with 1200 pixel and 1400 pixel width, the output list would be as shown in the
next example:

    (("00001.html", 1200) --> ("00001.html", 1400)) 

The Mapper outputs a list of key-value pairs `list(k1, v2)` with v2 being the text 
block width value and k1 the HTML page key simply repeated for each value. The 
value is a string with coordinates, representing width and height of the block 
element.

For each HTML page k1, the Reduce function aggregates all text block width values 
list(v2):

    reduce(k1, list(v2)) --> (k1,v3)

During this step it sums up the list(v2) text block width values and calculates 
the average width v3 (data type: org.apache.hadoop.io.LongWritable).

Related to the last example, this means that the Reduce function would calculate the 
average width like follows:

    (”00001.html”, (1200;1400)) --> (”00001.html”, 1300) 

Installation
------------

    cd hocr-parser-hadoopjob
    mvn install

Usage
-----

Execute hadoop job from the command line:

    hadoop jar target/hocr-parser-hadoopjob-1.0.jar-with-dependencies.jar 
      -d /hdfs/path/to/sequencefile/ -n job_name

References
----------

[1] Thomas M. Breuel, ”The hOCR Microformat for OCR Workﬂow and Results.”, 
    Proc. ICDAR, pg. 1063-1067. (2007).

[2] http://jsoup.org

[3] http://blog.cloudera.com/blog/2009/02/the-small-files-problem/
