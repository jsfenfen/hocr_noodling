What Word Where?
=============

The routines in this repo treat a page of scanned text as a geography, where a bounding box around each word is analogous to the boundaries of a geographic region. That's not an analogy--it actually relies on code developed for geographic information systems (GIS) applications. By entering a sample of a larger corpus of documents into a spatially-enabled database, users can create (and test) reasonable page identifiers (to choose pages of interest) and extraction templates (to locate specific sections of text to extract). Once the general extraction parameters are set, users can read through the full document set, applying the page identifiers and, when applicable, the extraction templates, without having to load any of the data into a database.

This library assumes data has already been extracted by optical character recognition (OCR) according to the [hOCR format](http://en.wikipedia.org/wiki/HOCR) using [tesseract](http://code.google.com/p/tesseract-ocr/).  HOCR outputs bounding boxes for words (and a whole hierarchy of sections of text; see more [here](https://docs.google.com/a/sunlightfoundation.com/document/d/1QQnIQtvdAC_8n92-LhwPcjtAUFwBlzE8EWnKAxlgVf0/preview).) 

This repo includes utilities for parsing pages from hOCR docs and uploading them in bulk to postgis/postgres. Also subroutines to return pages as an array of words with [GEOS](http://trac.osgeo.org/geos/)-style polygons (we're using [django's bindings to GEOS](https://docs.djangoproject.com/en/dev/ref/contrib/gis/geos/), so the shapes are actually GEOSGeometry objects).

The original use case for this library is a massive corpus of tax documents, which are released as image files and OCR-ed using [tesseract](http://code.google.com/p/tesseract-ocr/). Each tax form consists of many different "schedules" which contain various information. Regularly extracting a particular value requires first identifying all the pages it might appear on, writing a page identifier that locates those pages (e.g. a page that says "Schedule F" somewhere near the top left of the page) and then extracting a dollar figure at a particular position on that page. 


##### A few quirks:
We're assuming that hOCR arrives as latin-1 *despite* being labeled as utf-8. That's because the test data came from a version of tesseract with [this bug](https://groups.google.com/forum/#!topic/tesseract-ocr/UiyIMUWMzsU).

We're assuming the precise syntax generated by tesseract--other hOCR flavors may break this.

	

Requirements
============

Running the whole demo requires django running with a postgres database with postgis installed and working, as well as a variety of python libraries (see requirements.txt)

That said, the hocr page parser ( in hocr_util/parser/ ) should work without django/postgis -- see demo.py in that directory. At the moment, the stuff in display/ are something like first steps towards a raphael-based visualization of documents based on single-page hocr pages. It will currently display the location of word bounding boxes and the words if you hover on them, but not much more than that. There's a demo [here](http://jacobfenton.s3.amazonaws.com/hocr/display_hocr_page.html). 


Initial setup
=============
After running syncdb to create the initial models, we need to remove some checks that geodjango adds in on the assumption that we're using real geospatial data. Run:

	python manage.py drop_gis_constraints

For more details, see the command in documents/management/commands. 

Loading Test Data
===============

Sample hOCR documents are in the  /hocr_util/parser/test_hocr/ directory. (For reference the pdf documents they were parsed from are in /hocr_util/parser/test_pdf/ ).

Assuming all the requirements are there and syncdb has been run, and the troublesome database constraints habe been purged, you should be able to insert a document with 

	manage.py test_load

The command is in /documents/management/commands/ ; this isn't meant as a fuller loader--the file path is hardcoded--but an example of how to load. 

A fuller test data set--about 1,000 documents (~150MB zipped)--is available here: [http://pdf-liberation.s3.amazonaws.com/hocr_sample.zip](http://pdf-liberation.s3.amazonaws.com/hocr_sample.zip). To try loading the sample data, download and unzip the documents, then edit the file /documents/management/commands/load_hocr_sample.py by setting the SAMPLE_FILE_DIR path at the top of the file to the directory where the unzipped hocr files are located. Then try loading with:

	manage.py load_hocr_sample


To get a representation of a page with geometries included, see /documents/management/commands/test_word_shapes.py.

Envisioned Operation
============
This is, of course, experimental software, so we have no idea how it should work. But the envisioned use case is for a situation where there are millions of pages of hOCR data and dozens of different form submissions with both well-defined and open-ended fields. So that would likely entail training / testing and extraction. 

During the training / testing phase a substantial subset of the data is loaded into a postgis database and classifiers / extractors are written. The database is partially an investigative tool to be used interactively--show me all pages where there's a word near this position on the page that is 'Form' or something like that. Once reliable classifiers and extractors are written, the hope is that they could be applied one page at a time outside of a database context, thus alleviating the need to enter all of them into a database. 

One note about database entry: the routines included here are relatively efficient--they enter each page's worth of words in a single bulk copy statement--but constraints or indexes can slow this down substantially. Ultimately having one item per word is a fairly inefficient way of representing a large corpus, so… 

In extraction mode, pages would still be processed in the same way--an array of words with GEOSgeometry objects would returned. But the extractors and classifiers would have to be written to run on GEOSGeometries, not against postgis. There are, of course, queries possible in postgis that may not be supported by the geodjango geos api. 


	
