# Language-specific sorting #
If you want to sort strings which are not in English language, they may contain characters whose alphabetical order does not fit their character code order.

In these cases you can use the `Collator` class to gain a language specific comparator.

Let's assume we want to decide the alphabetical order of 2 Hungarian words:
```
    final Collator hunCollator = Collator.getInstance( new Locale( "hu" ) );
    
    final String hunWord1 = "elém";
    final String hunWord2 = "elemek";
    
    final int comparisionResult = hunCollator.compare( hunWord1, hunWord2 );
```