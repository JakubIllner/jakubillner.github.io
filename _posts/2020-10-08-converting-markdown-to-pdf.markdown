![Intro Picture](/images/2020-10-08-converting-markdown-to-pdf/lekarna-Kuks.JPG)

# __Introduction__

My team at Oracle decided to switch from Microsoft Word documents to Markdown.
The move is driven by the need to publish documents online, to achieve common
look-and-feel and quality for all documents, and to be able to version documents
easily.

Learning Markdown is extremely easy and there are many editors that support
preview of how Markdown would look when converted into HTML and published. The
formatting is applied automatically and consistently, without the hassle of
Word templates and styles. The documents are small and copy/paste between 
documents does not break formatting.

We also need to produce documents that are shared with customers and other
teams via email, Slack, or various content management systems. These documents
must have production quality and customers must not be able to change them,
as they typically represent our deliverables and other documents handed to
customers. We typically use PDF as ubiquitous format for such purpose.

In this post I describe a simple way of generating PDF from Markdown which 
leads to high quality PDF documents. It uses freely available tools pandoc
and wkhtmltopdf.

# __Tools__

In our scenario, we need to edit Markdown files and generate PDF locally on
laptops, running Windows, macOS, or Linux. I use Windows 10, so the setup
was tested on Windows platform, but it should work on Linux and Mac as well.

## __Markdown Editor__

As a MarkDown editor, I use UltraEdit from IDM ([https://www.ultraedit.com/](https://www.ultraedit.com/)).
This is a powerful, commercial text editor that I was using for many years.
It supports Markdown language, including preview of Markdown documents in the
built-in browser, using either predefined or custom CSS styles. Unfortunately,
it does not allow you to save the Markdown document as HTML.

## __pandoc__

Pandoc ([https://pandoc.org/](https://pandoc.org/)) is free, command line tool,
written by John MacFarlane, and released under the GPL licence. Pandoc supports
conversion between large number of markup formats, such as Markdown, HTML,
LaTeX, or Microsoft Word docx. On Windows, it can be installed using
installer for Windows.

## __wkhtmltopdf__

Wkhtmltopdf ([https://wkhtmltopdf.org/](https://wkhtmltopdf.org/)) is open
source command line tool, originally created by Jakob Truelsen, and released
under the LGPL license. Wkhtmltopdf renders HTML into PDF format using the Qt
WebKit rendering engine. On Windows, it can be also installed using
installer for Windows.

# __Markdown to PDF Conversion Steps__

I have converted Markdown document to PDF in steps outlined below.

![Markdown to PDF Conversion Steps](/images/2020-10-08-converting-markdown-to-pdf/markdown-to-pdf-conversion-steps.jpg)

Note that the process could be simplified - pandoc can generate PDF document
without explicitly calling wkhtmltopdf; also some Markdown editors support
conversion to HTML directly, without the need to use pandoc. However, I have 
found that these tools and process gives me the best control over the result.

## __Create CSS style__

Before writing Markdown documents and publishing them as HTML or PDF, it is 
useful to create (or download) a CSS style sheet document. CSS defines
all the formatting feature of the HTML document such as fonts, colors and
margins. Most of these features are used when generating PDF.

There are many publicly available CSS style sheets that can be used for this
purpose. Markdown editors come with their own CSS style sheets. But, as CSS
determines the visual clarity and representation and it is therefore important
part of the branding, you should standardize on your own CSS style.

## __Create Markdown document__

Create Markdown documents in your favourite Markdown editor, such as UltraEdit.
If available, use Preview functionality of the editor with the CSS style to
validate how the document will look like when published.

## __Convert Markdown to HTML__

Use pandoc to convert Markdown document to HTML.

```
pandoc <input>.md -o <output>.html -f gfm -t html -s --metadata title=<title> -c <style>.css
```

Parameter                     | Description
:--------                     | :----------
\<input\>.md                  | path to source Markdown document
-o \<output\>.html            | path to target HTML document
-c \<style\>.css              | path to CSS style sheet used for formatting the HTML
-f gfm                        | type of source document, GitHub flavour of Markdown in my case
-t html                       | type of target document 
-s                            | output will be standalone, valid HTML document with \<head\> and \<body\>
--metadata title="\<title\>"  | title of the document used in \<head\> of HTML document

Note that pandoc will use title not only in the \<head\> section, but it will
also create \<h1 class="title"\> in the \<body\> section. This is unwanted 
behaviour in my opinion, as I already have \<h1\> header at the top of the
Markdown document and it becomes duplicated. The workaround is to change the
CSS and not to display this class.

## __Convert HTML to PDF__

Use wkhtmltopdf to convert HTML document to PDF.

```
wkhtmltopdf ^
--enable-local-file-access ^
--outline ^
--images ^
--page-size A4 ^
--header-left "[doctitle]" ^
--header-right "<customer>" ^
--header-font-name "Segoe UI" ^
--header-font-size 10 ^
--header-line ^
--footer-left "[date] [time]" ^
--footer-right "[page]/[topage]" ^
--footer-font-name "Segoe UI" ^
--footer-font-size 10 ^
--footer-line ^
--log-level warn ^
<input>.html ^
<output>.pdf
```

Parameter                             | Description
:--------                             | :----------
--enable-local-file-access            | Allow wkhtmltopdf to access locally stored files
--outline                             | Generate PDF outline
--images                              | Include images into PDF
--page-size A4                        | Target paper size
--header-left "\[doctitle\]"          | Left aligned part of the header - populated with document title
--header-right "\<customer\>"         | Right aligned part of the header - e.g. customer name
--header-font-name "Segoe UI"         | Font used for header
--header-font-size 10                 | Size of font used for header
--header-line                         | Separate header from body by line
--footer-left "\[date\] \[time\]"     | Left aligned part of the body - populated with date and time
--footer-right "\[page\]/\[topage\]"  | Right aligned part of the body - populated with page number
--footer-font-name "Segoe UI"         | Font used for footer
--footer-font-size 10                 | Size of font used for footer
--footer-line                         | Separate footer from body by line
--log-level warn                      | Show warning and error messages only
\<input\>.html                        | path to source HTML document
\<output\>.pdf                        | path to target PDF document

Note that wkhtmltopdf may be called directly from pandoc. However, this method
supports only a subset of wkhtmltopdf's parameters. It does not support
outlines, headers, or footers. Because of these limitations I prefer calling
wkhtmltopdf explicitly.

# __Summary__

Using pandoc and wkhtmltopdf is powerful method for generating HTML and PDF documents
from Markdown. It can be easily scripted and extended beyond what I have shown here.
The most complex part is creating CSS style sheet, especially in case your requirements
go beyond simple CSS sheets.

You can also experiment with using LaTeX instead of HTML for even better quality of
generated PDF files. But, as LaTeX requires large footprint and specific skills, I
decided generating PDF from HTML is "good enough" for my purposes.


