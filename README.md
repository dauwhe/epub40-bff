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

We can describe everything we need to know about the bundle of documents that forms a publication with a relatively simple JSON structure. It consists of:

1. **metadata** about the publication as a whole.

2. A list of the **files** of the publication, along with information on their nature and sequence.

3. Optional **links** to related files or services, such as other formats, external metadata files, and search or syndication feeds.

4. Optional alternate representations of the publication ("multiple renditions" in EPUB-speak). 


## Fundamentals

Here's a simple example (we'll save the linked data for later):

```json
{
"metadata": {
  "title": "Moby-Dick",
  "identifier": "978031600000X",
  "language": "en",
  "modified": "2015-09-29T17:00:00Z"
},

"files": 
[  
  { "href": "toc.html", "mediaType": "text/html", "properties": "nav" },
  { "href": "c001.html", "mediaType": "text/html" },
  { "href": "c002.html", "mediaType": "text/html" },
  { "href": "style.css", "mediaType": "text/css" },
  { "href": "cover.jpg", "mediaType": "image/jpeg", "properties": "cover-image" }
]
}

```

### Files and sequences

You may have noticed that we don't have a separate "spine" and "manifest," in EPUB terms. We want to avoid as much duplication as possible, as well as the complications of @id and @idref. So we just have an array of links, which both list all the files, and determines the order of the content documents. We make the reasonable assumption that HTML files are part of the "spine", and that other files or not. You can override this with a boolean "sequence" key, if you want an image in the default sequence, or a content document out of the default sequence.

```json
{ "href": "p001.svg", "mediaType": "image/svg+xml", "sequence": "true" }

```

>**Note**: Filtering out the sequence from such a list can be done in one line of javascript by someone who hardly knows javascript. But it's a good example of making authoring simple, instead of optimizing for implementors. We could also define a metadata property which specifies the media-type of main-sequence files, to make it easier to produce an all-SVG publication. 


#### [A competing proposal](https://github.com/dauwhe/epub31-bff)

One could separate "spine" items from the rest of the manifest:

```json
{
  "metadata": {
    "title": "Moby-Dick",
    "identifier": "978031600000X",
    "language": "en",
    "modified": "2015-09-29T17:00:00Z"
  },

  "rendition": {

    "links": [
    {
      "href": "toc.html",
      "mediaType": "text/html",
      "properties": "nav"
    }, {
      "href": "c001.html",
      "mediaType": "text/html"
    }, {
      "href": "c002.html",
      "mediaType": "text/html"
    }],

    "manifest": {

      "links": [
        {
          "href": "style.css",
          "mediaType": "text/css"
        }, {
          "href": "cover.jpg",
          "mediaType": "image/jpeg",
          "properties": "cover-image"
        }
      ]
    }
  }
}

```

I feel this has some drawbacks:

1. Files exist at two different levels in the hierarchy, which I think is confusing. 

2. One cannot represent linear=no content documents. 

3. The structure is more complex and more nested, making hand-authoring or editing much more difficult.

4. The manifest key name behaves differently than in EPUB Classic. It is essentially the manifest minus the spine. 

5. It has been noted that the single-array version inappropriately orders non-spine content. But this is also true in this proposal. And ordering non-spine content could serve as a hint to user agents caching or rendering the content, putting critical content early.

6. In most cases, there's no informational difference between the proposals. In this proposal, you have to merge two arrays to get the classic manifest. In my proposal, you have to filter one array to get the classic spine. In either case that's one line of code, most likely. I think the benefits to authoring and understanding outweigh other concerns.


#### Aside: Generating a "spine" from the files array

```javascript
//create the spine from the files array
for(var i = 0; i < json.files.length; i++) {
// logic for what manifest files are in the spine
  if ( json.files[i].sequence == 'true' || ((json.files[i].type=='text/html' || json.files[i].type=='application/xhtml+xml') && json.files[i].sequence !== 'false' )) {
    var manifestItemRef = document.createElement('itemref');
    // create ID based on filename
    manifestItemRef.setAttribute('idref', 'a' + json.files[i].href.replace(/\.[^/.]+$/, "") );
    spine.appendChild(manifestItemRef);
    }
};
```


#### Even Simpler

If you use the proper file extension, any program that reads this manifest should be able to determine the correct mime type of the file. 


```json
{
"metadata": {
  "title": "Moby-Dick",
  "identifier": "978031600000X",
  "language": "en",
  "modified": "2015-09-29T17:00:00Z"
},

"files": 
[  
  { "href": "toc.html", "properties": "nav" },
  { "href": "c001.html" },
  { "href": "c002.html" },
  { "href": "style.css" },
  { "href": "cover.jpg", "properties": "cover-image" }
]
}

```

>**Note**: YAML might allow even simpler authoring, and the result easily converted to JSON:

```yaml
metadata:
  title: Moby-Dick
  language: en
  identifier: 9780316000000X
  modified: "2015-09-29T17:00:00Z"
# manifest and spine
files:
 - href: toc.html
   properties: nav
 - href: c001.html
 - href: c002.html
 - href: css/style.css
 - href: cover.jpg
   properties: cover-image

```
### Where would this JSON live?

>**Issue**: Should this be a standalone document? Should (or could) it be embedded in HTML to make the linked data within available to web crawlers?



### Multiple renditions

EPUB and OCF allow for alternate representations of a publication to be present in the same package. EPUB-BFF uses an `alternates` array to describe such renditions. Each alternate uses exactly the same syntax as the primary representation of the publication. Metadata in this context applies only to the particular representation, and may include what EPUB calls "rendition metadata".


```json
...
"alternates": [

    {
      "metadata": {
        "layout": "reflowable",
        "accessMode": "textual",
        "label": "Optimized for smaller screens"
      },
      "files": [
        {
          "href": "d001.html",
          "type": "text/html"
        },{
          "href": "d002.html",
          "type": "text/html"
        }
      ]
    }
    {
      "metadata": {
        "media": "color, min-width: 1920px",
        "layout": "pre-paginated",
        "accessMode": "visual",
        "label": "Color-optimized print replica"
      },
      "files": [
        {
          "href": "p001.html",
          "type": "text/html"
        },{
          "href": "p002.html",
          "type": "text/html"
        }
      ]
    }
  ]


```

### Links

We can also link to things outside our publication, or other formats.

```json
"links": [ 
  
  { 
    "href": "MobyDick.epub", 
    "mediaType": "application/epub+zip",
    "linkType": "alternate",
    "label": "EPUB version"
  }
]
}
```




####`linkType`

Here are some possibly useful link types. This is just a starting point. 

#####`alternate`

The `alternate` linkType describes a different manifestation of the same publication that exists outside the publication. It may be in a different language or a different format (such as "classic" EPUB). 

#####`metadata`

Used for linking to an external metadata record, such as ONIX or MARC. 

```json
  { 
    "href": "metadata.json", 
    "mediaType": "application/ld+json",
    "linkType": "metadata",
  }
```

#####`sequel`

Link to the next publication in a series.

#####`prequel`

Link to the previous publication in a series.

#####`search`

Points to a URL template which implements search in the publication:

```json
  { 
    "href": "/search?q={query}", 
    "mediaType": "text/html",
    "linkType": "search",
    "label": "Search this book",
    "templated": "true"
  }
```

#####`related`

The relation to the target publication is unspecified.

```json
 { 
    "href": "http://www.example.com/Typee/manifest.json", 
    "mediaType": "application/ld+json",
    "linkType": "related",
    "label": "Typee, another novel by Herman Melville",
  }
```


###Collections

Collections could be described as alternate representations of the publication. More TK.

###The Context and Linked Data

We can turn this JSON into JSON-LD linked data by adding a context, an `@id`, and a `@type`:

```json
{
  "@context": "http://idpf.org/2016/epub-bff-context.jsonld",
  "@id": "http://www.example.com/MobyDick/",
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
        idref
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


### Appendix: HTML-only version (just for fun)

We use the `nav` file to describe the content sequence, as well as providing metadata for the entire publication. 

#####Example
```html
<!DOCTYPE html>
<html lang="en" prefix="schema: http://schema.org/">
<head typeof="schema:Book">
<meta charset="utf-8" />
<title>Moby-Dick</title>
  <meta id="title" property="schema:name" content="Moby-Dick, or, The Whale">
  <meta id="pub-id" property="schema:isbn" content="9780000000001">
  <meta id="modified-date" property="schema:dateModified" content="2015-09-29T17:00:00Z">
  <meta id="language" property="schema:inLanguage" content="en-US">
  <!--list non-spine files here, to allow caching, etc-->
  <link href="style.css" type="text/css">
  <link href="cover.jpg" type="image/jpeg" rel="icon" sizes="any">
  <link href="whale.jpg" type="image/jpeg">
  <link href="boat.svg" type="image/svg+xml">
  <link href="notes.html" type="text/html" title="Notes from the editor">
  <link href="#nav" type="text/html" rel=contents>
  <link href="map.svg" type="image/svg+xml">
  <link href="c001.html" type="text/html">
  <link href="c002.html" type="text/html">
</head>
<body>
  <nav role="doc-toc" id="nav"> 
    <ol>
      <li> <a href="#nav" type="text/html">Contents</a> </li>
      <li> <a href="map.svg" type="image/svg+xml">Map</a> </li>
      <li> <a href="c001.html" type="text/html">Looming</a> </li>
      <li> <a href="c002.html" type="text/html">The Spouter-inn</a> </li>
    </ol>
  </nav> 
</body>
</html>

```

####Issues with HTML serialization

1. Vocabulary is not easily extended

2. Nesting/wrapping is difficult in html head

3. I've been unable to have a single schema.org object in both head and body

4. Unclear how to define multiple renditions

5. Much of the information is not created for human consumption



