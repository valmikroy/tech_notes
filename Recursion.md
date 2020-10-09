# Recursion 



Recursion can be divided into direct and indirect recursion.

### Tail and Head recursion 

```ruby
def tail_recursion(n)
    
  if n > 0
    puts "number #{n}"
    tail_recursion(n-1)  # keep taking action first and then wind up with nothing
  end
    
end

def head_recursion(n)
    
  if n > 0
    head_recursion(n-1)  
    puts "number #{n}" # reach till the end of the stack and then wind up
  end
    
end


tail_recursion(10) # this will print 10 to 1 in decrement 

head_recursion(10) # this will print 1 to 10 in the increment 
```

- Time Complexity For Recursion: O(n)
- Space Complexity For Recursion: O(n) - this can be imporoved by using while loop.





### Tree recursion 



Multiple recursion get spun up in the single run of the function. Permutation problem is the classic recursion.

```ruby
string=['A','B','C']


def permute(input,idx=0,output=[])
    
    output.push(input.clone) if idx == input.length
    (idx..input.length-1).each do |i|
      input[idx],input[i]  = input[i], input[idx]
      permute(input,idx+1,output)  
      input[idx],input[i]  = input[i], input[idx]
    end

    return output
end

p permute(string)
```











