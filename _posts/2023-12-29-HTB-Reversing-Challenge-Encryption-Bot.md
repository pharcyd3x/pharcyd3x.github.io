---
tags: [posts]
title: "HTB Reversing Callenge - Encryption Bot"
excerpt: "Walkthrough for HackTheBox Encryption Bot reversing challenge"
---

The challenge files for "Encryption Bot" include a binary named "chal" and "flag.enc" which contains a string that seems to be an encrypted flag "9W8TLp4k7t0vJW7n3VvMCpWq9WzT3C8pZ9Wz". Initial analysis of **chal** shows it is a stripped 64-bit ELF executable. Running strings for low hanging fruit did not show much info other than a bogus flag and "rm data.dat", which could indicate i/o actions taking place within the binary. 

<img src="{{ site.url }}{{ site.baseurl }}/assets/images/Selection_272.png" alt="">


Continuing looking for some more basic info about the binary, ltrace and strace were used and resulted in some useful information. Running **ltrace** showed that the binary does indeed conduct some file operations as a call to fopen was found: ```fopen("data.dat","r")```. Additionally a call to **strlen** is observed before we get an error message telling us that the length of our input was incorrect, indicating the flag must be a specific length: ```strlen("asdfadsf") & puts("I'm encrypt only specific length of character")```. strace confirmed the same info. 

<img src="{{ site.url }}{{ site.baseurl }}/assets/images/Selection_273.png" alt="">


Before loading the binary into a disassembler I find it helpful to take note of what I've learned about the file that may help narrow down what I'm looking for once I'm in there, so far we know the following:
1. Binary is stripped 64-bit ELF
2. prompts user for string to encrypt
3. content in "flag.enc" is likely the encrypted output of real flag
4. flag must be a specific length
5. a file/stream with "data.dat" may be involved in the encryption process

----

Note: I am only a few months into RE and have solely used Ghidra, honestly only chose Ghidra bc it's free, and based on youtube videos seemed way more intuitive to me than IDA. I am nowhere close to being a Ghidra expert, or even an experienced Ghidra user, so if I explain something wrong or miss something, my b - just using it how I know how off my limited experience and it has worked for me so far lol,  tips or criticisms on twitter are welcome if ya feel the need to do so.  

Since the binary is stripped we have to find the "main" function, which can easily be found in the ```_libc_start_main()``` call at the entry point (```_libc_start_main() is responsible for initilzaing the execution envrionemnt and handling the calling of main()```). The first argument passed to _libc_start_main() can be assumed to be the main function. 

Entry point:

<img src="{{ site.url }}{{ site.baseurl }}/assets/images/Selection_274.png" alt="">


main / FUN_0010163e()

<img src="{{ site.url }}{{ site.baseurl }}/assets/images/Selection_275.png" alt="">

After inspecting main() we can confirm we're in the right place by inspecting its contents. First FUN_00101628() prints the banner that is displayed when running the executable, next you see the prompt the user is shown to enter input, as well as a check to see if "data.dat" exists on the system already. After that FUN_001015Dc() is ran which was determined to be the function that checks the length of the user input.
  
<img src="{{ site.url }}{{ site.baseurl }}/assets/images/Selection_277.png" alt="">

With the info we have so far I have cleaned up variable/function names to make it easier to proceed

<img src="{{ site.url }}{{ site.baseurl }}/assets/images/Selection_278.png" alt="">

As we can see from the screenshot above, once the input_length check has succeeded (input was 27 chracters) we reach the next two functions which we can assume is where the actual input manipulation happens. First we see FUN_0010131d(user_input) - stepping into this function you will see that the function loops through each character in the input string and passes it to a function I renamed "temp_file_write()". 

I've outlined with comments where each step takes place, but what temp_file_write() does is take each character of the input string, convert it to it's binary representation and then write that binary representation **backwards** into a file stream - "data.dat". This is the first time in the program we see our input string being manipulated. If you don't understand how a character is converted to binary with this code block, you can look up binary conversion algorithms for C. 

<img src="{{ site.url }}{{ site.baseurl }}/assets/images/Selection_279.png" alt="">

We have now identified some key steps to the encoding process, here's what we have so far:
1. Input must be 27 characters long
2. Take each character and convert it to binary 
3. take that binary no., reverse it, and then write it to file stream

----

If we go back to main() we have one more function we haven't analyzed that most likely will complete the encoding process, FUN_001014ba(). This last function can look complex but when broken down piece by piece is rather simple(ish). First it loops through the "data.dat" file where the reversed binary no. of our input is stored, this file will be 217chars long (217 = 8bits * 27chars). The most important part of this function is the last, where it runs a function on every index which is divisible by 6 (```if ((index_original !=0) && (index_original % 6 == 0) ```).  This is how it works:

For every index value divisible by 6, run 6 iterations of index - 1 (loop backwards), where file_completion_2(index2) is run. Let's say we have a list of 12 items:
[0,1,2,3,4,5,6,7,8,9,10,11] -> the condition ```if ((index_original !=0) && (index_original % 6 == 0) ``` would first hit on "5" or list[6] as the index is not 0 and divisible by 6. The function file_completion_2() would be ran on list[6],list[5],list[4]... down to list[0]. The second hit would be on "11" or list[12] and the same would happen: file_completion_2() ran for 6 iterations on list[12] through list[7]

<img src="{{ site.url }}{{ site.baseurl }}/assets/images/Selection_282.png" alt="">

What file_completion_2() does is essentially 2^index power - if the index passed is 6 it will multiply 2, 6 times. That result is returned and multiplied by the value from our binary_list (either a 0 or 1). 

<img src="{{ site.url }}{{ site.baseurl }}/assets/images/Selection_281.png" alt="">

When I first encountered this challenge, this part threw me off. Continuing to try and write what I understand so far step by step, even if I don't see the end picture yet helps. So far our updated encoding steps are: 
1. Input must be 27 characters long
2. Take each character and convert it to binary 
3. take that binary, reverse it, and then write it to file stream
4. Iterate through reversed binary and on every index divisible by 6 (this would happen a total of 36 times - 217/6):
	- Take character from that index and the 5 before it (i.e 011010) and convert that to a number (conversion happens through file_completion_2 and lines following)
5. Take that result_number and pass it to print_enc_input(result_number)

Now even though the result_number to pass can't be obtained by us without knowing the correct value of the reversed binary no., we have enough to potentially work backwards. The function print_enc_input() is the last part of the program. This function is simple as it will simply take the result_number and move <result_number> of positions away from the starting char "R" in the string:
"RSTUWYZ0123456789ABCDEFGHIJKLMNOPqrstuvwxyz"

<img src="{{ site.url }}{{ site.baseurl }}/assets/images/Selection_283.png" alt="">

---

Okay, we have reached the end of the encoding algorithm and have our final steps:
1. Input must be 27 characters long
2. Take each character and convert it to it's binary no. (27chars * 8 = 217 total)
3. take that binary, reverse it, and then write it to file stream (217bits)
4. Open file stream and iterate through reversed binary and on every index divisible by 6 (this would happen a total of 36 times - 217/6):
	- Take character from that index and the 5 before it (i.e 011010) and convert that to a number (conversion happens through file_completion_2 and lines following)
5. Take that result_number and pass it to print_enc_input(result_number) -> Returns the character that is present "result_number" of spots away from "R" in the string "RSTUWYZ0123456789ABCDEFGHIJKLMNOPqrstuvwxyz"
6. Step 5 will run 36 times (see step 4)-> producing a 36 character encoded string.

Knowing these steps and the end result of these steps - the string in "flag.enc" challenge file - "9W8TLp4k7t0vJW7n3VvMCpWq9WzT3C8pZ9Wz", the next step is to use all the information we have to write a script to decode flag.enc. This part was the hardest for me, conceptualizing how to work backwards given the encoding steps, it took some time but through trial and error I was able to figure it out. Being that this challenge is still active I won't include the full decoding script, but the steps I used to write it. If these steps aren't enough to help you write the script (wasn't for me at first either) I suggest encoding your own 27 character string with the program and then work backwards using it's output. These are the decryption steps I used for my solution:

1. Starting with string "9W8TLp4k7t0vJW7n3VvMCpWq9WzT3C8pZ9Wz" -> find out each characters index value compared to "RSTUWYZ0123456789ABCDEFGHIJKLMNOPqrstuvwxyz"
	- Example: the first char "9" is the 18th index in "RSTUWYZ0123456789ABCDEFGHIJKLMNOPqrstuvwxyz"
	- Result should be an array of 36 integers that represent index vals
2. For every index divisible by 6 - convert the value at that index and the previous 5 to its binary no. 
	- Result should be array of binary no. representation of each value from the array from step 1. (should be 217 chars)
3. Now we have our binary representation of our encrypted str, but we need to convert it back. Using the inverse method of the way the program originally converted the string to binary, we can convert ours back. 
	- Result should be the flag! 

