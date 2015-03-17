# How to use regular expressiosn #

If we have a regular expression, the easiest way to use it is:
```
    final String regexp = "<some regexp comes here>";
    // Cache the pattern so we don't have to parse it again and again
    final Pattern pattern = Pattern.compile( regexp );
    
    final String tryMe = "some text";
    
    if ( pattern.matcher( tryMe ).matches() )
        ; // Do something
```

# Regular Expressions #

```
    // Simplified email pattern:
    final String emailRegexp = "\\S+@\\S+\\.\\S+";
```