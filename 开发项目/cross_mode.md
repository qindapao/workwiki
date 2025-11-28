# cross_mode

Whether an element can be crossed should be an attribute of the element, not an asciio global mode. Just like whether to allow automatic connecting and boundary connection, whether it can cross should essentially be an element attribute.

```perl
sub is_border_connection_allowed { 0 }
sub allow_border_connection { ; }
sub is_autoconnect_enabled { my ($self) = @_ ; return ! $self->{AUTOCONNECT_DISABLED} ; }
sub enable_autoconnect { my ($self, $enable) = @_ ; $self->{AUTOCONNECT_DISABLED} = !$enable ; }
```

The biggest problem with the global cross mode is that in the same diagram, we may hope that some parts need to be crossed, but other parts do not need to be crossed at all. If you force the crossover, it will not only consume more CPU, but also not what the user expects. Therefore, the crossover of elements should be left to the user, rather than a global enforcement mode.

Another problem with the global cross mode is that some parts of the chart need to be crossed to express the original meaning, but users may turn off the cross mode by mistake, which will also lead to confusion and misunderstanding.

## crossing situations

Only elements that are allowed to cross need to process their cross over layers when drawing and exporting. This is divided into three situations.

1. allow cross + not allow cross     => Does not handle crossovers
2. not allow cross + not allow cross => Does not handle crossovers
3. allow cross + allow cross         => Handle crossovers

## Areas that require special consideration

When grouping and ungrouping strip groups, attention should be paid to correctly handling the crossover of internal elements. Then whether the strip group itself can cross with other elements is also determined by its own attributes. It has nothing to do with the cross properties of the inner elements.

## Operations required

1. Elements add the attribute of whether they can cross.
2. Controls the cross property switching of one and multiple elements.
3. Provide a global tag. If the tag is turned on, the newly inserted elements will have the cross attribute by default; the tag is off by default, and the newly inserted elements will have no cross attributes.
4. We can customize the cross style of elements, provide a right-click drop-down list or keyboard shortcut switching.

A note on the last point, for example, the cross character is a `+` , but in a circuit diagram we might want it to be a `(` or `)` .

```txt
         |    |     |
         |    |     |
      ---+----)-----(---
         |    |     |
         |    |     |
```

## Other optimizations

The algorithm can be optimized into the following algorithm, which eliminates the need to use cache, saves memory, and has the same level of speed.

```txt
New cross point calculation algorithm.                                      
Use 3 bits to represent the attributes of chars in each direction.          
none        : 0                                                             
ascii       : 1                                                             
thin        : 2                                                             
bold        : 3                                                             
double      : 4                                                             
dotted      : 5                                                             
bold dotted : 6                                                             
                                                                            
Other bits can be used to specify other attributes of the character,        
such as whether it is an arrow                                              
A 32-bit integer can precisely represent all attributes of a character.     
                                                                            
                                                                            
           non text flag(1: non text)                                      
            │border char flag(1: border char)                                                                                         
            │ │line char flag(1: line char)                                
            │ │ │arrow flag(1: arrow)                                      
            │ │ │ │                                                        
            v v v v┃ 315 ┃ 225 ┃ 135 ┃ 45  ┃right┃left ┃down ┃ up          
   │3│3│2│2│2│2│2│2┃2│2│2┃2│1│1┃1│1│1┃1│1│1┃1│1│ ┃ │ │ ┃ │ │ ┃ │ │ │       
   │1│0│9│8│7│6│5│4┃3│2│1┃0│9│8┃7│6│5┃4│3│2┃1│0│9┃8│7│6┃5│4│3┃2│1│0│       
───┼─┼─┼─┼─┼─┼─┼─┼─╂─┼─┼─╂─┼─┼─╂─┼─┼─╂─┼─┼─╂─┼─┼─╂─┼─┼─╂─┼─┼─╂─┼─┼─┼───    
  ┼│ │ │ │ │ │ │ │0┃ │ │ ┃ │ │ ┃ │ │ ┃ │ │ ┃ │1│ ┃ │1│ ┃ │1│ ┃ │1│ │       
  ┤│ │ │ │ │ │ │ │0┃ │ │ ┃ │ │ ┃ │ │ ┃ │ │ ┃ │1│ ┃ │ │ ┃ │1│ ┃ │1│ │       
  ╫│ │ │ │ │ │ │ │0┃ │ │ ┃ │ │ ┃ │ │ ┃ │ │ ┃ │1│ ┃ │1│ ┃1│ │ ┃1│ │ │       
  >│ │ │ │ │ │ │ │1┃ │ │ ┃ │ │ ┃ │ │ ┃ │ │ ┃ │ │1┃ │ │ ┃ │ │ ┃ │ │1│       
  <│ │ │ │ │ │ │ │1┃ │ │ ┃ │ │ ┃ │ │ ┃ │ │ ┃ │ │ ┃ │ │1┃ │ │ ┃ │ │1│       
  +│ │ │ │ │ │ │ │0┃ │ │ ┃ │ │ ┃ │ │ ┃ │ │ ┃ │ │1┃ │ │1┃ │ │1┃ │ │1│       
   │ │ │ │ │ │ │ │ ┃ │ │ ┃ │ │ ┃ │ │ ┃ │ │ ┃ │ │ ┃ │ │ ┃ │ │ ┃ │ │ │       
                                                                           
To determine what character a crosspoint should be, I should take the      
neighbors in the eight directions of this crosspoint. Then, I need to look 
at the properties of these eight characters. The judgment logic is as      
follows:                                                                   
  1. the char above takes the properties of the part below.                
  2. The char to the left takes the properties of the part to the right.   
  3. The char below takes the properties of the part above.                
  4. The char to the right takes the properties of the part to the left.   
  5. The char at 45 degrees takes the properties of the 225-degree part.   
  6. The char at 135 degrees takes the properties of the 315-degree part.  
  7. The char at 225 degrees takes the properties of the 45-degree part.   
  8. The char at 315 degrees takes the properties of the 135-degree part.                      
In this way, after extracting the character attributes from the eight      
directions and performing bitwise operations, we can concatenate them into 
a new complete 32-bit integer. By looking up a table, we can determine the 
character corresponding to this new integer (if it's 0, then it's an empty 
character, indicating no intersecting characters). This allows us to obtain
the character at the center point. There's no need for complex             
calculations or caching.                                                   
```

