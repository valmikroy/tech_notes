# System Design - Remote boot in the wild



Design a box which you ship to customer and customer just plug it to the network and from that point onwards you control most of the maintaince of the host while giving customer desired functionality for the data transfer.



#### Why?

- We designing this system for mass data transfer from customers place to ours because of various real world constraints on the cloud. 

  - security 
  - accessiblity through internet over remote locations 
  - requires post processing on the premise


- We lower overhead for the customer to maintain.


#### How?
- We will ship a box with operating system and all the software - most secure box ever. Big HTTP packet sent by road. (availble and managable ) 
- Box should connect back to our infrastructure as soon as it shows up over the network. (availble and managable ) 
- Customer should get some kind of API to do data transfer to the box. (availble and managable )
- Easiest mode of troubleshooting (scalable and reliable) 
- Quicker the transfer is better (efficient)

#### What?



Process

- Lay out the process for customer in simple steps.
- Box boots up and wait to get an IP. If it get registerd then control path should guide customer through all the next steps.
- If registration fails then we need have a failsafe option.
- We should also have failure gathering mechanism which will allow us to improve our process in future.
- We might ask customer to tell us static IP information before shipping this box.
- We can also keep a probe device handy which we can send in customers premise in advance so we can have better information about their network topology.



Tech steps

- We will ship the box with the pre-loaded OS image.
- Connecting to the network
   - Provide mechanism to the customer to connect this box to the network.
   - connect this back to our cloud as soon as it gets the network access.
   - provide indicator if it fails to reach the cloud and provide rudimentry troubleshooting steps. This is where we have to be very precise about what going to happen during failure states
      - Internet gateway is not reachable
      - Networking is not up
      - Cloud itself is down? 
- Let customer autodiscover box through provided utility to which it can transfer those files. May be use of mDNS which we are not sure how will work on the remote network.
- OS need to be harden in terms of security so it box is unbreakable.
- communication between client utility for copying and the box must be absolute secure. Credentials can be delivered through the cloud.
- For credentials you need to have shared secret on the box. Or we have to install public key which was a part of this box.
- encrypt the disk completely. Some kind of TPM device has to be installed.
- Create a control path from the box to the cloud so we can provide
   - ability update/delete data through UI.
   - Other infromation and health statistics of the box.







  

  



