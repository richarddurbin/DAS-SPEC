# **DAS: The Dagstuhl Assembly Standard (V 1.0)**

*J. Chin, R.Durbin, G. Myers*  
*Aug. 31, 2016*

## PROLOG

DAS is a generalization of GFA that allows one to specify an assembly graph in either less detail,
e.g. just the topology of the graph, or more detail, e.g. the multi-alignment of reads giving
rise to each sequence.  We further designed it to be a suitable representation for a string
graph at any stage of assembly, from the graph of all overlaps, to a final resolved assembly
of contig paths with multi-alignments.  Apart from meeting our needs, the extensions also
supports other assembly and variation graph types.

We futher want to emphasize that we are proposing a core standard.  As will be seen later in
the technical specification, the format is **extensible** in that additional description lines
can be added and additional SAM tags can be appended to core description lines.

In overview, an assembly is a graph of vertices called **segments** representing sequences
that are connected by **edges** that denote local alignments between the vertex sequences.
At a minimum one must specify the length of the sequence represented, further specifying the
actual sequence is optional.  In the direction of more detail, one can optionally specify a
collection of external sequences, called **fragments**, from which the sequence was derived (if
applicable) and how they multi-align to produce the sequence.  Similarly, the specification
of an edge need only describe the range of base pairs aligned in each string, and optionally
contain a **trace** or a **CIGAR string** to describe the alignment of the edge.  Traces are a
space-efficient Dazzler assembler concept that allow one to efficiently reconstruct an
alignment in linear time, and CIGAR strings are a SAM concept explicitly detailing the
columns of an alignment.  Many new technologies such a Hi-C and BioNano maps organize segments
into scaffolds along with traditional data sets involving paired reads, and so a “gap” edge
concept is also introduced so that order and orientaiton between disjoint contigs of an
assembly can be described.

## GRAMMAR

```
<spec>     <- <header> ( <segment> | <edge> | <gap> | <group> )+

<header>   <- H VN:Z:1.0 {TS:i:<trace spacing>}

<segment>  <- S <sid:id> <slen:int> <sequence>

<fragment> <- F <sid:id> [+-] <external:id>
                  <sbeg:int> <send:int> {CL:B:<pbeg>,<pend>,<plen>} {<alignment>}

<edge>     <- E <eid:id> <sid1:id> [+-] <sid2:id>
                         <beg1:pos> <end1:pos> <beg2:pos> <end2:pos> {<alignment>}

<gap>      <- G <eid>:id> <sid1:id> [+-] <sid2:id> <dist:int> <var:int>

<group>    <- P[UO] <name:id> <item>([ ]<item>)*

    <id>        <- [!-~]+
    <item>      <- <sid:id> | <eid:id>
    <pos>       <- ^ | <int> | $
    <sequence>  <- * | [A-Za-z]+
    <alignment> <- TR:B:i<trace array>
                   CG:Z:<CIGAR string>

      <CIGAR string> <- ([0-9]+[MX=ID])+
      <trace array>  <- <int>(,<int>)*
```

In the grammar above all symbols are literals other than tokens between <>, the derivation
operator <-, and the following marks:

  * {} enclose an optional item
  * | denotes an alternative
  * * zero-or-more
  * + one-or-more
  * [] a set of one character alternatives.

Like GFA, DAS is tab-delimited in that every lexical token is separated from the next
by a single tab.

Each descriptor line must begin with a letter and lies on a single line with no white space
before the first symbol.   The tokens that generate descriptor lines are \<header\>, \<segment\>,
\<fragment\>, \<edge\>, \<gap\>, and \<group\>.
Any line that does not begin with a recognized code (i.e. H, S, F, E, G, or P) can be ignored.
This will allow users to have additional descriptor lines specific to their special processes.
Moreover, the suffix of any DAS descriptor line may contain any number of user-specific SAM
tags which are ignored by software designed to support the core standard.  These user-specific
tags must occur after any of the optional SAM tags in the core specification above
(i.e. VN, TS, TR, CG, and CL).  

## SEMANTICS

The header just contains a version number, 1.0, and if Dazzler trace points are used to
efficiently encode alignments, then one needs to specify the trace point spacing with a
TS tag in the header.

A segment is specified by an S-line giving a user-specified ID for the
sequence, its length in bases, and the string of bases denoted by the segment or * if absent.
The length does not need to be the actual length of the sequence, if given, but is rather
an indication to a drawing program of how long to draw the representation of the segment.

Fragments, if present, are encoded in F-lines that give (a) the segment they belong to, (b) the
orientation of the fragment to the segment, (c) an external ID that references a sequence
in an external collection (e.g. a database of reads or segments in another DAS or SAM file),
and (d) the interval of the vertex segment that the external string contributes to.  One can
optionally give a trace or CIGAR string detailing the alignment, and optionally specify, should
clipping/trimming the fragment sequence be necessary, what portion of the fragment is used and
its length with a CL-tag.

Edges are encoded in E-lines that in general represent a local alignment between arbitrary
intervals of the sequences of the two vertices in question. One gives first an edge ID and
then the segment ID’s of the two vertices and a + or – sign between them to indicate whether
the second segment should be complemented or not.  An edge ID does not need to be unique or
distinct from a segment ID, *unless* you plan to refer to it in a P-line (see below).  This
allows you to use something short like a single *-sign as the id when it is not needed.

A CIGAR string (or trace) describing the alignment is optional, but
one must give the intervals that are aligned as a pair of positions where a position can have
the special value ^ denoting the beginning of the segment, or the special value $ denoting
the end of the segment.  An edge may optionally be given a unique ID in the case that
a user needs to explicitly refer to it in a group line (see below).  This is specified by
optionally giving the ID prefixed with an @-sign at the start of the line.

The DAS concept of edge generalizes the link and containment lines of GFA.  For example a GFA
edge which encodes what is called a dovetail overlap (because two ends overlap) is simply a DAS
edge where end1 = $ and beg2 = ^ or beg1 = ^ and end2 = $.   A GFA containment is
modeled by the case where beg2 = ^ and end2 = $ or beg1 = ^ and end1 = $.  The figure
below illustrates:

![Fig. 1](DAS.Fig1.png)

Special codes could be adopted for dovetail and containment relationships but the thought is
there is no particular reason to do so, the use of the special characters for terminal positions
makes their identification simple both algorithmically and visually, and the more general
scenario allows interesting possibilities.  For example, one might have two haplotype bubbles
shown in the “Before” picture below, and then in a next phase choose a path through the
bubbles as the primary “contig”, and then capture the two buble alternatives as a vertex
linked with generalized edges shown in the “After” picture.  Note carefully that you need a
generalized edge to capture the attachment of the two haplotype bubbles in the “After” picture.

![Fig. 2](DAS.Fig2.png)
 
While one has graphs in which vertex sequences actually overlap as above, one also frequently
encounters models in which there is no overlap (basically edge-labelled models captured in a
vertex-labelled form).  This is captured by edges for which beg1 = end1 and beg2 = end2 (i.e.
0-length overlap)!

While not a concept for pure DeBrujin or long-read assemblers, it is the case that paired end
data and external maps often order and orient contigs/vertices into scaffolds with
intervening gaps.  To this end we introduce a “gap” edge described in G-lines that give the
estimated gap distance between the two vertex sequences and the variance of that estimate
or 0 if no estimate is available.

A group encoding on a P-line allows one to name and specify a subgraph of the overall graph.
Such a collection could for example be hilighted by a drawing program on
command, or might specify decisions about tours through the graph.  The P is immediately
follwed by U or O indicating either an *unordered* or *ordered* collection.  The remainder of
the line then consists of a name for the collection followed by a non-empty list of ID's
refering to segments and/or edges that are *separated by single spaces* (i.e. the list is
in a single column of the tab-delimited format).  An unordered
collection refers to
the subgraph induced by the vertices and edges in the collection (i.e. one adds all edges
between a pair of segments in the list and one adds all segments adjacent to edges in the
list.)   An ordered collection captures paths in the graph consisting of the listed objects
and the implied adjacent objects between consecutive objects in the list (e.g.
the edge between two consecutive segments, the segment between two consecutive edges, etc.)

Note carefully that there may be several edges between a given pair of segments, so in
in this event it is necessary that edges have unique IDs that can be referred to as a
segment pair does not suffice.  The convention is that every edge has an id, but this
id need only be unique and distinct from the set of segment ids when one wishes to refer
to the edge in a P-line.  So an id in a P-line refers first to a segment id, and if there
is no segment with that id, then to the edge with that id.  All segment id's must be
unique, and it is an error if an id in a P-line does not refer to a unique edge id
when a segment with the id does not exist.
