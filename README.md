[![Build Status](https://travis-ci.org/antigenomics/vdjdb-db.svg?branch=master)](https://travis-ci.org/antigenomics/vdjdb-db)

# VDJDB: A curated database of T-cell receptor sequences of known antigen specificity

This repository hosts the submissions to database and scripts to check, fix and build the database itself.

To build database from submissions, go to ``src`` directory and run ``groovy -cp . BuildDatabase.groovy`` script (requires [Groovy](http://www.groovy-lang.org/)).

To query the database for your immune repertoire sample(s) use the [VDJdb](https://github.com/antigenomics/vdjdb) software.

A web-based GUI for the database [VDJdb-server](https://github.com/antigenomics/vdjdb-server) is currently under development.

## Submission guide

To submit previously published sequence follow the steps below:

* Create an issue(s) labeled as ``paper`` and named by the paper pubmed id, ``PMID:XXXXXXX``. Note that if paper is a meta-study, you can mark it as ``meta-paper`` and link issues for its references in a reply to this issue. Also note that in case submitting unpublished sequences, choose any appropriate issue name with details on submitter (name, organization, etc) in issue comments.

* Create new branch and add chunk(s) for corresponding papers named as ``PMID_XXXXXXX``. Don't forget to close/reference corresponding issues in the commit message.

* Create a pull request for the branch and check if it passes the CI build. If there are any issues, modify them by fixing/removing entries as necessary.

The ``BuildDatabase`` routine ran during CI tests upon each submission and prior to every database release implements table format checks, CDR3 sequence checks and fixes (if possible), and VDJdb confidence score assignment (see below).

To view the list of papers that were not yet processed follow [here](https://github.com/antigenomics/vdjdb-db/labels/paper).

An XLS template is available [here](https://raw.githubusercontent.com/antigenomics/vdjdb-db/master/template.xls).

> **CAUTION** make sure that nothing is messed up (``x/X`` frequencies are transformed to dates, bad encoding, etc) when importing from XLS template. The format of all fields is pre-set to *text* to prevent this case.

## Database specification

Each database submission in ``chunks/`` folder should have the following header and columns:

### Complex information columns (required)

These columns convey full information about TCR:peptide:MHC complex and are mandatory for any submission.

column name     | description
----------------|-------------
cdr3.alpha | TCR alpha CDR3 amino acid sequence. Complete sequence starting with C and ending with F/W should be provided if possible. Trimmed sequences will be fixed at database building stage in case sufficient V/J germline parts are present
v.alpha | TCR alpha Variable (V) segment id, up to best resolution possible (``TRAVX*XX``, e.g. ``TRAV7``, ``TRAV7*01``, ``TRAV7*02``...). 
j.alpha | TCR alpha Joining (J) segment id
cdr3.beta | TCR beta CDR3 amino acid sequence
v.beta | TCR beta V segment id
d.beta | TCR beta Diversity segment id
j.beta | TCR beta J segment id
species | TCR parent species (``HomoSapiens``, ``MusMusculus``,...)
mhc.a | First MHC chain allele, to best resolution possible, ``HLA-X*XX:XX``, e.g. ``HLA-A*02:01``
mhc.b | Second MHC chain allele (``B2M`` for MHCI)
mhc.class | ``MHCI`` or ``MHCII``
antigen.epitope | Amino acid sequence of the epitope
antigen.gene | Parent gene of the epitope sequence (e.g. ``pp24``)
antigen.species | Parent species of the antigen, to the best clade resolution possible (e.g. ``HIV-1``, ``HIV-1*HXB2``)
reference.id | Pubmed id, doi, etc
submitter | Name of submitting person/organization

> **Notes:**

> In case given record represents a clonotype with either TCR alpha or beta sequence unknown, missing CDR3/V/(D)/J fields should be left blank.

> V/(D)/J fields can be left blank, however this will abrogate CDR3 fixing/verification procedure for a given record.

> Any record should have at least one of CDR3 alpha/beta fields that are not blank.

### Method information columns (optional)

Optional columns (i.e. it is not required to fill them, but they **should** be present in table header) that ensure correct confidence ranking of a given entry. Used to calculate a single confidence score based on various factors, e.g. fraction of a given TCRab sequence among tetramer+ clones sequenced and verification experiments performed.

column name     | description
----------------|-------------
method.identification | ``tetramer``, ``dextramer``, ``pelimer``, ``pentamer``, etc for sorting-based identification. For molecular assays use: ``antigen-loaded-targets``, ``antigen-expressing-targets``, or ``other``
method.frequency | Frequency in isolated antigen-specific population, reported as ``X/X``. E.g. ``7/30`` if a given V/D/J/CDR3 is encountered in 7 out of 30 tetramer+ clones
method.singlecell | ``yes`` if single cell sequencing was performed, blank otherwise
method.sequencing | Sequencing method: ``sanger``, ``rna-seq`` or ``amplicon-seq``
method.verification | ``tetramer``, ``dextramer``, ``pelimer``, ``pentamer``, etc for methods that include TCR cloning and re-staining with multimers. ``antigen-loaded-targets``, ``antigen-expressing-targets`` for molecular assays. Several comma-separated verification methods can be specified.

> **Notes:**

> In case ``method.identification`` is left blank, the record is automatically assigned with a lowest confidence score possible.

> For special cases such as CD8-null tetramers that utilize HLA with mutated residues that abrogate CD8 binding, specify ``cd8null-tetramer`` in ``method.identification`` field rather than using ``mhc.a`` field.

During database build phase, the information from columns mentioned above is collapsed to a JSON string and stored in a single ``method`` column, e.g.: 
```json
{
   "identification":"tetramer-sort",
   "frequency":"5/13",
   "sequencing":"sanger",
   "verification":"antigen-loaded-targets"
}
```

### Meta-information columns (optional)

column name     | description
----------------|-------------
meta.study.id | Internal study id
meta.cell.subset | T-cell subset, free style, e.g. ``CD8+``, ``CD4+CD25+``
meta.subset.frequency | Frequency of a given TCR sequence in specified cell subset, e.g. ``5%`` means that the TCR sequence represents an expanded clone occupying 5% of CD8+ cells
meta.subject.cohort | Subject cohort, free style, e.g. ``healthy``, ``HIV+``,...
meta.subject.id | Subject id (e.g. ``donor1``, ``donor2``,...)
meta.replica.id | Replicate sample coming from the same donor, also applies for different time points, etc (e.g. ``5mo``)
meta.clone.id | T-cell clone id
meta.epitope.id | Epitope id (e.g. ``FL10``)
meta.tissue | Tissue used to isolate T-cells: ``PBMC``, ``spleen``,... or ``TCL`` (T-cell culture) if isolated from re-stimulated T-ells
meta.donor.MHC | Donor MHC list if available, blank otherwise
meta.donor.MHC.method | Donor MHC typing method if available, blank otherwise
meta.structure.id | PDB structure ID if exists, or blank
comment | Plain text comment, maximum 140 characters

> **Note:**

> While these columns are optional, subject identifier, replica identifier, etc are used when scanning submission for duplicates. Normally duplicate records (with identical **complex information** columns) are not allowed, but they will not be considered as duplicates in case they have distinct id fields mentioned above.

During database build phase, the information from columns mentioned above is collapsed to a JSON string and stored in a single ``meta`` column, e.g.: 
```json
{
   "cell.subset":"CD8+",
   "subject.cohort":"HSV-2+",
   "subject.id":12,
   "clone.id":46,
   "tissue":"PBMC"
}
```

## Database processing

### CDR3 sequence fixing

At this stage, a series of checks is performed for CDR3 sequence and reported V/J segments:

* In case of *canonical* (starting with conserved ``C`` and ending with ``F/W``) CDR3 sequences: checks if 5' and 3' germline parts match corresponding V/J segment sequences.
* In case of truncated CDR3 sequences: adds conserved ``C/F/W`` residues. Can add more missing residues in case a relatively large contiguous V/J germline match is present.
* In case excessive germline part is reported (e.g. ``FGXG`` instead of simply ``F`` at CDR3 3' part), excessive residues are removed.
* Can correct mismatches in V/J germline regions in case a reliable non-contiguous V/J match is found.

The main reason behind that is that current immune repertoire sequencing (RepSeq) data processing software reports *canonical* clonotype sequences, high number antigen-specific TCR sequences present in literature are reported inconsistently. The latter greatly complicates annotation of RepSeq data using known antigen-specific TCR sequences.

In case of good V/J germline matching and errors in CDR3 sequence, the final CDR3 sequence in the database is replaced by its fixed version. The following report of CDR3 fixer is placed under ``cdr3fix.alpha`` and ``cdr3fix.beta`` columns, e.g.

```json
{
	"fixNeeded":true,
	"good":false,
	"cdr3":"CASSQDVGTGGVFALYF",
	"cdr3_old":"CASSQDVGTGGVFALY",
	"jFixType":"FixAdd",
	"jId":"TRBJ1-6*01",
	"jCanonical":true,
	"jStart":14,
	"vFixType":"FailedBadSegment",
	"vId":null,
	"vCanonical":true,
	"vEnd":-1
	}
```

and

```json
{
	"fixNeeded":true,
	"good":true,
	"cdr3":"CASSLSRGGNQPQYF",
	"cdr3_old":"CASSLSRGGNQPQY",
	"jFixType":"FixAdd",
	"jId":"TRBJ1-5*01",
	"jCanonical":true,
	"jStart":9,
	"vFixType":"NoFixNeeded",
	"vId":"TRBV14*01",
	"vCanonical":true,
	"vEnd":4
}
```

Field descriptions:

field | description
------|-------------
``fixNeeded`` | ``true`` if corrected CDR3 sequence differs from the original one, ``false`` otherwise
``good`` | ``true`` if the fix can be applied, ``false`` if the fix cannot be applied due to bad V/J entry or no V/J matching
``cdr3`` | Fixed CDR3 sequence
``cdr3_old`` | Original CDR3 sequence
``jFixType`` | Type of fix applied to CDR3 J germline part
``jCanonical`` | ``true`` if CDR3 ends with ``F`` or ``W``, ``false`` otherwise
``jId``  | J segment identifier
``jStart``  | A 0-based index of first CDR3 amino acid just before J segment
``vFixType`` | Type of fix applied to CDR3 V germline part
``vCanonical`` | ``true`` if CDR3 starts with ``C``, ``false`` otherwise
``vId`` | V segment identifier
``vEnd``  | A 0-based index of the first CDR3 amino acid after V segment

> **Note:**

> Possible V and J fix types: ``NoFixNeeded``, ``FixAdd``, ``FixReplace``, ``FixTrim``, ``FailedReplace`` (too many mismatches), ``FailedBadSegment`` (bad segment entry), ``FailedNoAlignment`` (no alignment at all)

### VDJdb scoring

At the final stage of database processing, TCR:peptide:MHC complexes are assigned with confidence scores. Scores are computed according to reported **method** entries. First, a score is assigned to identification method. In case a given complex was identified using multimer sorting, frequency of a given TCR sequence among sorted population is taken into account. Additional score points are assigned in case one or more verification steps are reported for a given entry.

Max score is then selected among different records (independent submissions, replicas, etc) pointing to the same unique complex entry (i.e. set of unique **complex** fields).

> **Note:** A record that has a ``meta.structure.id``, i.e. a structural data associated with it, automatically gets the highest VDJdb score possible.

score | description
------|----------------------
0     | No data
1     | Low-confidence
2     | Medium-confidence
3-6   | High-confidence
7     | Has structural data
