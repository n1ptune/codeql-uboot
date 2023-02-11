##step-3
query a function named strlen
```
import cpp

from Function f
where f.getName() = "strlen"
select f, "a function named strlen"
```
##step-4
query a function named memcpy
```
import cpp

from Function f
where f.getName() = "memcpy"
select f, "a function named memcpy"
```
##step-5
query macros named ntohs or ntohl or ntohll
```
import cpp

from Macro macro
where macro.getName() = "ntohs"
    or macro.getName() = "ntohl"
    or macro.getName() = "ntohll"
select macro, "found macro"
```
more effective
```
import cpp

from Macro macro
where macro.getName() in ["ntohs", "ntohl", "ntohll"]
select macro, "found macro"
```
use Regular Expression
```
import cpp

from Macro macro
where macro.getName().regexpMatch("ntoh(s|l|ll)")
select macro, "found macro"
```
##step-6
query the caller of a function
```
import cpp

from FunctionCall fc
where fc.getTarget().getName() = "memcpy"
select fc, "caller of the memcpy"
```
##step-7
query the invocations of macros
```
import cpp

from MacroInvocation mi
where mi.getMacro().getName().regexpMatch("ntoh(s|l|ll)")
select mi
```
##step-8
query the expressions that correspond to macro invocations.
```
import cpp

from MacroInvocation mi
where mi.getMacro().getName().regexpMatch("ntoh(s|l|ll)")
select mi.getExpr()
```
##step-9
Write your own CodeQL class to represent a set of interesting source code elements
To define a class, you write:
1. The keyword class.
2. The name of the class. This is an identifier starting with an uppercase letter.
3. The supertypes that the class is derived from via extends and/or instanceof
4. The body of the class, enclosed in braces.
```
class OneTwoThree extends int {
    OneTwoThree() { // characteristic predicate
      this = 1 or this = 2 or this = 3
    }
  
    string getAString() { // member predicate
      result = "One, two or three: " + this.toString()
    }
  
    predicate isEven() { // member predicate
      this = 2
    }
}
```
```
import cpp

/**
 * An expression involved when swapping the byte order of network data.
 * Its value is likely to have been read from the network.
 */
class NetworkByteSwap extends Expr {
  NetworkByteSwap() {
    exists(MacroInvocation mi |
      mi.getMacroName().regexpMatch("ntoh(s|l|ll)") and
      this = mi.getExpr()
    )
  }
}

from NetworkByteSwap n
select n
```
##step-10
query to track the flow of tainted data from network controlled interges to the memcpy length argument
```
import cpp
import semmle.code.cpp.dataflow.TaintTracking
import DataFlow::PathGraph

/**
 * An expression involved when swapping the byte order of network data.
 * Its value is likely to have been read from the network.
 */
class NetworkByteSwap extends Expr {
    NetworkByteSwap() {
      exists(MacroInvocation mi |
        mi.getMacroName().regexpMatch("ntoh(s|l|ll)") and
        this = mi.getExpr()
      )
    }
  }

class Config extends TaintTracking::Configuration {
    Config() { this = "no matter" }
    
    override predicate isSource(DataFlow::Node source) {
        source.asExpr() instanceof NetworkByteSwap
    }
    
    override predicate isSink(DataFlow::Node sink) {
        exists(FunctionCall fc | fc.getTarget().getName() = "memcpy" and sink.asExpr() = fc.getArgument(2))
    }
}

from Config cfg, DataFlow::PathNode source, DataFlow::PathNode sink
where cfg.hasFlowPath(source, sink)
select sink, source, sink, "Network byte swap flows to memcpy"
```