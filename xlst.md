**XSL** stands for eXtensible Stylesheet Language, and **XSLT** for XSL Transformations. 

## Installing

sudo yum install libxslt-python.x86_64

Don't know why it, but it installs `xsltproc`. 

`xsltproc [stylesheet] [file1]`

## XSLT cheatsheet

### Variable

```xml
  <xsl:variable name="email">
     <xsl:value-of select="owner"/>@editure.com.au
  </xsl:variable>

   <a href="mailto:{$email}"><xsl:value-of select="$email"/></a>
```

### If expression

```xml
<xsl:if test="test expression">
</xsl:if>
```

## Links

1. http://linux.dd.com.au/wiki/XSLT_Tutorial
