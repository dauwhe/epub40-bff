#EPUB-BFF: Simpler, Friendlier, JSONlier.

EPUB as it exists today is not directly usable by a web browser. The web-friendly content files are inside a zip package, which also contains container and package files expressed in a custom XML vocabulary.

The goal of a browser-friendly format (henceforth EPUB-BFF) is to make it easier for web developers to display EPUB content by [1] allowing an unzipped ("exploded") publication, and [2] by providing an alternative serialization of the information in container.xml and the package document(s).


##Introduction: The Browser-Friendly Format (aka "BFF")

EPUB exists because the web doesn't allow us to easily speak about
collections of documents. Web documents can link to each other, and link
relations let you say a few things about what's on the other end of a
link. But you can't say two documents are part of a larger entity. You
can't say this metadata applies to a group of documents.

EPUB has filled that gap with the package file, which includes publication metadata, a list of files that make up the publication ("manifest"), and information about their ordering ("spine"). However, there's a lot of duplication and indirection involved in the XML, and we believe a simpler conceptual model is possible. 

We can describe everything we need to know about the bundle of documents that forms a publication with a relatively simple JSON file. It consists of:

1. Metadata.

2. A list of the files of the publication, along with information on their nature and sequence.

3. Optional links to related files, such as alternate renditions or translations, other formats (like "classic" EPUB), previews, and so on. One can also link to services such as search or syndication feeds, or external metadata files.  

##What is a publication?

EPUB is limited because all the components of a publication must be packaged together. EPUB is unnecessarily complex because many of the extension specs try to incorporate multiple publications into the same package. 

But does this make sense in a browser-friendly world? Might it be better to link than to bundle? Adopting this approach in BFF opens up some very interesting possibilities.


We can define a publication not as a bundle of files, but merely as the JSON object that describes the relationships between the constituent documents, which could be anywhere. So instead of a multiple-rendition EPUB, we can have multiple JSON files that each describe a single rendition, but link to each other. 


##Example

Here's a simple example:

```json
{
"metadata": {
  "title": "Moby-Dick",
  "identifier": "978031600000X",
  "language": "en-US",
  "modified": "2015-09-29T17:00:00Z",
  "version": "1.0"
},

"files": 
[  
  { "href": "c001.html", "mediaType": "text/html", "properties": "nav" },
  { "href": "c002.html", "mediaType": "text/html" },
  { "href": "c003.html", "mediaType": "text/html" },
  { "href": "style.css", "mediaType": "text/css" },
  { "href": "cover.jpg", "mediaType": "image/jpeg", "properties": "cover-image" }
]
}

```

### Files and sequences

You may have noticed that we don't have a separate "spine" and "manifest," in EPUB terms. We want to avoid as much duplication as possible, as well as the complications of @id and @idref. So we just have an array of links, which both list all the files, and determines the order of the content documents. We make the reasonable assumption that HTML files are part of the "spine", and that other files or not. You can override this with a boolean "sequence" key, if you want an image in the default sequence, or a content document out of the default sequence.

```json
{ "href": "p001.svg, "mediaType": "image/svg+xml", "sequence": "true" }

```

### Alternate renditions

For most publications, we won't need anything else. But if you want to offer multiple versions of a publication, previews, etc. then we add a `links` array:

```json
"links": [ 
  { 
    "href": "rendition-es.json", 
    "mediaType": "application/epub+json",
    "linkType": "alternate",
    "label": "Spanish version",
    "lang": "es"
  },
  { 
    "href": "rendition-fr.json", 
    "mediaType": "application/epub+json",
    "linkType": "alternate",
    "label": "Version française",
    "lang": "fr"
  },
  { 
    "href": "/search?q={query}", 
    "mediaType": "text/html",
    "linkType": "search",
    "label": "Search this book",
    "templated": "true"
  },
  { 
    "href": "MobyDick.epub", 
    "mediaType": "application/epub+zip",
    "linkType": "alternate",
    "label": "EPUB version"
  },
  { 
    "href": "preview.json", 
    "mediaType": "application/epub+json",
    "linkType": "preview",
    "label": "Preview"
  },
  { 
    "href": "metadata.json", 
    "mediaType": "application/ld+json",
    "linkType": "metadata",
  }
]
}
```

####`linkType`

#####`alternate`

The `alternate` linkType describes a different manifestation of the same publication. It may be in a different language (as the first two above) or a different format (the fourth here is a regular EPUB version of the publication). 

#####`preview`

This describes a subset of the publication intended as a preview. 

#####`metadata`

Used for linking to an external metadata record, such as ONIX or MARC. 

#####`sequel`

Link to the next publication in a series.

#####`prequel`

Link to the previous publication in a series.

#####`related`

The relation to the target publication is unspecified.




####The Context and Linked Data

We can turn this JSON into JSON-LD linked data by adding a context and a `@type`:

```json
{
  "@context": "http://idpf.org/2016/epub-bff-context.jsonld",
  "@type": "http://schema.org/Book",
  "metadata": {
    "title": "Moby-Dick"...

```

Here's the context itself:

```json
"@context": {
  "title": "http://schema.org/name",
  "language": "http://schema.org/inLanguage",
  "version": "http://schema.org/version",
  "modified": "http://schema.org/dateModified",
  "mediaType": "http://schema.org/fileFormat",
  "href": "http://schema.org/url"
}
```



###Appendix: Ontology for "Classic EPUB"

Just to help think about this stuff...

```
publication
	unique identifier
	release identifier
	rendition+ (default rendition)
		package document
			metadata
				type?
					dictionary
					education/edupub??
						teacher-edition
						teacher-guide
					preview
					distributable-object
					index
			manifest
				publication resource
					core media type resource
						epub content document
							xhtml content document
								epub navigation document
								scripted content document
							svg content document
							fixed layout content document
					foreign resource
			spine
			collection
				metadata
					role
						dictionary
						distributable-object
						index
						preview
						index-group
						scriptable-component
						manifest
				link
				collection
```






