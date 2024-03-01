# [Writing a simple 16 bit VM in less than 125 lines of C](https://www.andreinc.net/2021/12/01/writing-a-simple-vm-in-less-than-125-lines-of-c#add---adding-two-values)
```shell
git clone git@github.com:nomemory/lc3-vm.git
```
> enviroment : 
>  1. code is written in C11
>  2. Based on [LC-3 Computer Architecture](https://en.wikipedia.org/wiki/Little_Computer_3)

* Target : 在本文結束時，我們將擁有一個基於暫存器的工作虛擬機，能夠解釋和執行一組有限的 ASM 指令以及一些額外的程式來測試一切是否正常。

* [LC-3指令集 指令/狀態碼介紹](https://www.twblogs.net/a/5ea653306052e13757c1c334)
* 進階 : [Write your Own Virtual Machine](https://www.jmeiners.com/lc3-vm/) by [Justin Meiners](https://github.com/justinmeiners) and [Ryan Pendleton](https://github.com/rpendleton).
* another programming language(Rust) implement : [check this out](https://ptomato.wordpress.com/2022/01/10/a-little-computer/)
## VM
* System Virtual Machines : 提供real machine的完整替代品。它們<mark>實現了足夠的功能</mark>，允許作業系統在它們上運行。它們<mark>可以共享和管理硬件</mark>，有時多個環境可以在同一台實體機器上運行而不會互相妨礙。
* Process Virtual Machines : 它們更簡單，旨在在與平台無關的環境中執行電腦程式(ex. JVM)
* 為簡單起見，我們特意從以下功能中剝離了 LC-3 實作：中斷處理、優先權、行程、狀態暫存器 (Program Status Register, PSR)、特權模式、管理程式堆疊、使用者堆疊。我們將只虛擬化最基本的硬件，並且我們將透過陷阱與外界（stdin、stdout）進行互動。
## Von Neumann Model
![graph](https://i0.wp.com/semiengineering.com/wp-content/uploads/2018/09/Screen-Shot-2017-04-26-at-1.08.57-PM.png?ssl=1)

* **CPU** is divided into three layers : **ALU**, **CU**, and **Registers**
* **ALU** : 代表實際執行數據指令的電路（ex. ADD, XOR, Division, etc.) 
* **CU** : 協調 CPU 上的活動
* **Register** : 暫存器是位於 CPU 層級的可快速存取的 **"slots"**。 ALU 對暫存器進行操作。它們的數量很少(這是一個相對的說法，因為它取決於架構)，因此 CPU 內可以載入的資料量是有限的。我們使用暫存器與Main Memory互動。典型的場景包括將記憶體位置載入到暫存器中，執行一些更改，然後將資料放回記憶體中。
* **Main Memory** : 主記憶體被想像為一個由 W 個 words 組成的擴展 “array”，每個 words 有 N 位元。程式指令和相關資料以二進位格式儲存在主記憶體中。每個記憶體字包含一條指令或程式資料(例如，用於計算的數字)
* **Input/Output device** : 使電腦能夠與外界進行通訊
## Implementing the VM
* VM functions like this:
    1. Load the program into the main memory
    2. In the `RPC` register, keep the current instruction that we need to execute
        > `RPC` : return program counter
    3. Obtain the Operation Code(first 4 bits) from the instruction and based on that, we decode the rest of the parameters
    4. Execute the method associated with the given instruction
    5. Increment `RPC` and continue with the next instruction

![flow chart](https://www.andreinc.net/assets/images/2021-12-01-writing-a-simple-vm-in-less-than-125-lines-of-c/vm.drawio.png)
## Main Memory
* W = UINT16_MAX, N = 16 bits
    * W : 主記憶體中總共有多少個 entry
    * N : 每個 entry 的大小是多少個 bits

    ```c
    uint16_t PC_START = 0x3000;
    uint16_t mem[UINT16_MAX+1] = {0};
    ```
* UINT16_MAX=65535, 所以此 system 無法 run/load 超過 65535 條指令的程式
* 將 `0x3000`以下的空間保留給其他 potential components使用(ex. toy operating system)
* 儘管可以直接讀取或寫入main memory(因為沒有額外的映射機制), 但未來若要添加額外的邏輯(映射)或對記憶體存取施加驗證時,會較不方便,所以添加了兩個 function, for reading(mr(...)), for writing(mw(...))
    ```c
    static inline uint16_t mr(uint16_t address) { return mem[address];  }
    static inline void mw(uint16_t address, uint16_t val) { mem[address] = val; }
    ```
## Registers
* 共有 10 個 registers, 每個 size 為 16 bits
    * `RO` : general-purpose register. 用來 reading/writing data from/to `stdin`/`stdout`
    * `R1`, `R2`, ...`R7` : general-purpose registers
	* `RPC` : program counter register. 保存下一個將要執行指令的記憶體位置
	* `RCND` : conditional register. 此 flag 保存前一次 
    ```c
    enum regist {R0 = 0, R1, R2, R3, R4, R5, R6, R7, RPC, RCND, RCNT};
    uint16_t reg[RCNT] = {0};
    ```
    > example : access the R3.
    > ```c
    > reg[R3] = ...;
    > ```
## Instructions
* Instruction format
    ![instruction format](https://www.andreinc.net/assets/images/2021-12-01-writing-a-simple-vm-in-less-than-125-lines-of-c/instr.drawio.png)
    * first 4 bits for OP code
    * remaining 12 bits for params
*  根據 OP code , 了解如何從剩下的 bits 中 "decode"/"extract" 其餘參數
	* 擷取 OP code
		```c
		#define OPC(i) ((i)>>12)
		```
* 由與 OP code 僅 4 個 bits(指令數最多為16), 我們將所有指令保存在 array 中, 其 value 為指向 function 的 pointer
	```c
	#define NOPS (16) // number of instructions
	typedef void (*op_ex_f)(uint16_t instruction);
	//
	// ... other operations here
	//
	static inline void add(uint16_t i)  { /* code here */ }
	static inline void and(uint16_t i)  { /* code here */ }
	//
	// ... other operations here
	//
	op_ex_f op_ex[NOPS] = { 
		br, add, ld, st, jsr, and, ldr, str, rti, not, ldi, sti, jmp, res, lea, trap 
	};
	```
* 此 ASM(Assembly Language) 為複製 LC3 規範中的指令

    | Instructin | Op Code Hex | Op Code Bin | C function | Comments |
    | :--------: | :---------: | :---------: | :--------: | :------: |
    | br | 0x0 | 0b0000 | void br(uint16_t i) | Conditional branch |
    | add | 0x1 | 0b0001 | void add(uint16_t i) | Used for addition |
    | ld | 0x2 | 0b0010 | void ld(uint16_t i) | Load RPC + offset |
    | st | 0x3 | 0b0011 | void st(uint16_t i) | Store |
    | jsr | 0x4 | 0b0100 | void jsr(uint16_t i) | Jump to subroutine |
    | and | 0x5 | 0b0101 | void and(uint16_t i) | Bitwise logical AND |
    | ldr | 0x6 | 0b0110 | void ldr(uint16_t i) | Load Base + offset |
    | str | 0x7 | 0b0111 | void str(uint16_t i) | Store Base + offset |
    | rti | 0x8 | 0b1000 | void rti(uint16_t i) | <mark>Return from interrupt(not implemented)</mark> |
    | not | 0x9 | 0b1001 | void not(uint16_t i) | Bitwise complement |
    | ldi | 0xA | 0b1010 | void ldi(uint16_t i) | Load indirect |
    | sti | 0xB | 0b1011 | void sti(uint16_t i) | Store indirect |
    | jmp | 0xC | 0b1100 | void jmp(uint16_t i) | Jump/Reture to subroutine |
    | Unused | 0xD | 0b1101 | NULL | Reserved OP code |
    | lea | 0xE | 0b1110 | void lea(uint16_t i) | Load effective address |
    | trap | 0xF | 0b1111 | void trap(uint16_t i) | System trap/call |

    > [JSR, Jump to SubRoutine](https://blog.csdn.net/kkk584520/article/details/41775367)

    > [What's the purpose of the LEA instruction? - Stack overflow](https://stackoverflow.com/questions/1658294/whats-the-purpose-of-the-lea-instruction)

* 根據上述指令, 可以分成以下五種主要的類型
	* br, jmp, jsr : 用於程式的控制流程 : 從一條指令跳轉到另一條(類似於 goto) 或條件跳轉 (類似於 if)
	* ld, ldr, ldi, lea : 將數據從 main memory 加載到暫存器中
	* st, str, sti : 將數據從暫存器中存回至 main memory
	* add, and, not : 對暫存器保存的數據進行(數學)運算
	* trap : 特殊指令, 與鍵盤交互(讀取字符或數字), 並在 stdout 上打印訊息

>[!CAUTION]
>可以進一步練習實現, XOR、除法、乘法, 以進一步了解ASM

* `RCND` : 共三個狀態
	> 我們有 RCND 的原因是為了幫助我們進行分支。例如，我們想看看一個數字 a 是否比另一個數字 b 大。我們可以計算它們的差值，如果結果是負數，則RCND為1<<2。然後我們可以使用 br 指令跳到另一個指令，就像在高階程式語言中使用 IF 語句一樣

	* `1<<0` (P from positive) : if the last operation yielded a positive resule
	* `1<<1` (Z from zero) : if the last operation yielded 0
	* `1<<2` (N from Negative) : if the last operation yielded a negative result
	* Example : 
		```c
		enum flags { FP = 1 << 0, FZ = 1 << 1, FN = 1 << 2 };

		static inline void uf(enum regist r){
			if(reg[r] == 0) reg[RCND] = FZ; // the value in r is zero
			else if(reg[r] >> 15) reg[RCND] = FN; // the value in r is z negative number
			else reg[RCND] = FP; // the value in r is a positive number
		}
		```
## Function Implement
### add - Adding two values
* 使用 bit[5] 來表示使用的 add 版本, 0 for add1, 1 for add2
* The first one (add1) is used for adding the values of two registers: `SR1`, `SR2`, and storing their sum in `DR1`
	![$add^1$](https://www.andreinc.net/assets/images/2021-12-01-writing-a-simple-vm-in-less-than-125-lines-of-c/add.drawio.png)
* The second one (add2) is used to add a “constant” value (IMM5) to `SR1`, and store the result in `DR1`
	![$add^2$](https://www.andreinc.net/assets/images/2021-12-01-writing-a-simple-vm-in-less-than-125-lines-of-c/add2.drawio.png)

> `DR`, *Destination Register*

> `SR`, *Source Register*

* `IMM5` 是一個正數或負數(共5 bits), 必須編寫一個擴展符號(sign)的函數, 使其與 16bits 格式相容
	```c
	// sign extend (sext)
	#define SEXTIMM(i) sext(IMM(i), 5)

	// if the bth bit of n is 1(number is negative), fill up with 1s the remaining bits else return the number as it is
	static inline uint16_t sext(uint16_t n ,int b){
		return ((n >> (b-1))&1) ? (n | (0xFFFF << b)) : n;
	}
	```
	* Example : 
		```c
		uint16_t a = 0x16;          // The 5th bit is 1
									// This means that the number kept in the last 5 bits
									// is negative.
									// So it's important to store it as correctly in a uint16_t type
									// In this regard we apply SEXTIMM(a) to make it happen

		fprintf_binary(stdout, a);
		fprintf(stdout, "\n");

		fprintf_binary(stdout, SEXTIMM(a));
		fprintf(stdout, "\n");

		// Output
		//
		//  0000 0000 0001 0110 <--- a in binary
		//  1111 1111 1111 0110 <--- SEXTIMM(a) in binary
		```
* 取得第 5 個 bit
	```c
	#define FIMM(i) ((i>>5)&1)
	```
* 取得 DR(store the add output)
	```c
	#define DR(i) (((i)>>9)&0x7)
	```
* 取得 SR1(參數）
	```c
	#define SR1(i) (((i)>>6)&0x7)
	```
* 取得 SR2(參數）
	```c
	#define SR2(i) ((i)&0x7)
	```
* 取得 IMM5(const value)
	```c
	#define IMM(i) ((i)&0x1F)
	```
* Add function
	```c
	static inline void add(uint16_t i){
		reg[DR(i)] = reg[SR1(i)] + (FIMM(i) ? SEXTIMM(i) : reg[SR2(i)]);
		// update the conditional register depending on the value of DR1
		uf(DR(i));
	}
	```
### and - Bitwise logical AND
* 使用 bit[5] 來表示使用的 and 版本, 0 for and1, 1 for and2
* The first one (and1) applies binary & on the values of two registers: `SR1`, `SR2`, and storing the result in `DR1`
	![$and^1$](https://www.andreinc.net/assets/images/2021-12-01-writing-a-simple-vm-in-less-than-125-lines-of-c/and.drawio.png)
* The second one (and2) applies binary & between `SR1` and `IMM5` and stores the result in `DR1`
	![$and^2$](https://www.andreinc.net/assets/images/2021-12-01-writing-a-simple-vm-in-less-than-125-lines-of-c/and2.drawio.png)
* And function
	```c
	static inline void and(uint16_t i){
		reg[DR(i)] = reg[SR1(i) & (FIMM(i) ? SEXTIMM(i) : reg[SR2(i)]);
		uf(DR(i));
	}
	```
### ld - Load RPC + offset
* 將數據從 main memory 載入到目標暫存器(DR1)
* 通過向`RPC`暫存器加上偏移量獲得位置, 將 `RPC` 作為一個 referencing point

![ld operate](https://www.andreinc.net/assets/images/2021-12-01-writing-a-simple-vm-in-less-than-125-lines-of-c/ldexp.drawio.png)
> So let’s say the RPC points to the memory address: 0x3002. If the offset is set to 100 we just read the data from location 0x3002+100==0x3066 and load it in the destination register (in our case R4, but this can be everything).

* instruction format

	![instruction format](https://www.andreinc.net/assets/images/2021-12-01-writing-a-simple-vm-in-less-than-125-lines-of-c/ld.drawio.png)


* ld function
	```c
	#define P0FF9 sext((i)&0x1FF, 9)
	static inline void ld(uint16_t i){
		reg[DR(i)] = mr(reg[RPC] + POFF9(i));
		uf(DR(i));
	}
	```

>[!CAUTION]
>由於 offset 僅9 bits, 意味著偏移量最多可容納的最大整數為 $2^9 -1 = 511$。因此, 根據程式的儲存位置(儲存方式), `ld`指令將無法訪問某些內存區域(查看以下圖片), 為了解決上述問題, 我們使用 `ldi` 指令解決

![example](https://www.andreinc.net/assets/images/2021-12-01-writing-a-simple-vm-in-less-than-125-lines-of-c/ldexpnma.drawio.png)

### ldi - Load indirect
* 將數據載入暫存器, 使用 中間地址(intermediary address) 訪問<mark>far-away</mark>的內存位置
	![](https://www.andreinc.net/assets/images/2021-12-01-writing-a-simple-vm-in-less-than-125-lines-of-c/ldiexp.drawio.png)
	
	> So let’s say the RPC points to 0x3002. Just like before, we use a 9 bit offset to access another memory address at position (RPC+offset). In our case, the offset=100, so the memory address we read is 0x3066.

	> But instead of loading (directly) 0x3066 into DR, we look at the value contained by 0x3066, which is 0x3204. We now, bring the value of 0x3204 inside the DR.

* instruction format
	
	![](https://www.andreinc.net/assets/images/2021-12-01-writing-a-simple-vm-in-less-than-125-lines-of-c/ldi.drawio.png)


* function format
	```c
	static inline void ldi(uint16_t i){
		// perform two memory reads
		reg[DR(i)] = mr(mr(reg[RPC]+POFF9(i)));
		uf(DR(i));
	}
	```
>[!CAUTION]
>這並不表示 `ldi` 比 `ld` 更好, 因為 `ldi` 需要執行兩次讀取, 但能夠存取到更遠的 address 

### ldr - Load Base + offset
* 與從 `RPC` 開始的 `ld` 指令相比, `ldr` 使用不同的 Base address(保存在暫存器中的內存地址)
* instruction format

	![](https://www.andreinc.net/assets/images/2021-12-01-writing-a-simple-vm-in-less-than-125-lines-of-c/ldr.drawio.png)
* To extract `OFFSET6`
	```c
	#define POFF(i) sext((i)&0x3F, 6)
	```
* function format
	```c
	static inline void ldr(uint16_t i){
		reg[DR(i)] = mr(reg[SR1(i)] + POFF(i));
		uf(DR(i));
	}
	```
### lea - load effective address
* 將內存地址加載到暫存器中, 與`ld`、`ldi`和`ldr`相比, `lea`並不將程序數據帶入暫存器, 而是將內存地址帶入暫存器

	![](https://www.andreinc.net/assets/images/2021-12-01-writing-a-simple-vm-in-less-than-125-lines-of-c/leaexp.drawio.png)
	> Let’s say the RPC points to 0x3002. The offset9=100, so we load into the DR register the result of 0x3002+100=0x3066.

* instruction format :
	
	![](https://www.andreinc.net/assets/images/2021-12-01-writing-a-simple-vm-in-less-than-125-lines-of-c/lea.drawio.png)

* function format : 
	```c
	static inline void lea(uint16_t i){
		reg[DR(i)] = reg[RPC] + POFF9(i);
		uf(DR(i));
	}
	```
### not - Bitwise complement
* 對 `SR1` 執行 bitwise complement, 並儲存至 `DR1`
* instruction format : 
	
	![](https://www.andreinc.net/assets/images/2021-12-01-writing-a-simple-vm-in-less-than-125-lines-of-c/not.drawio.png)
* function format :
	```c
	static inline void not(uint16_t i){
		reg[DR(i)] = ~reg[SR1(i)];
		uf(DR(i));
	}
	```
### st - Store
* 將給定暫存器的值儲存到內存位置

	![](https://www.andreinc.net/assets/images/2021-12-01-writing-a-simple-vm-in-less-than-125-lines-of-c/stexp.drawio.png)
	>Let’s say RPC is pointing to 0x3002, and SR refers to R1=0x0001. The st instruction will write the value of R1 to RPC+offset.

* instruction format :
	
	![](https://www.andreinc.net/assets/images/2021-12-01-writing-a-simple-vm-in-less-than-125-lines-of-c/st.drawio.png)

* function format : 
	```c
	static inline void st(uint16_t i){
		mw(reg[RPC] + POFF9(i), reg[DR(i)]);
	}
	```
>[!CAUTION]
>1. 與 `ld` 相同, `st`也受到內存尋址能力的限制, 為此, 引入 `sti` 指令
>2. 無須更新任何flag(uf), 因為沒有對暫存器進行運算(加減乘除)

### sti - store indirect
* 不直接寫入內存地址, 而是使用其作為中間寫入地址。這個中間地址包含要寫入的實際內存地址
	![](https://www.andreinc.net/assets/images/2021-12-01-writing-a-simple-vm-in-less-than-125-lines-of-c/stiexp.drawio.png)
	> To write the content of R1=0x0001 (SR), to 0x3204, we will first have to read the memory location RPC+offset = 0x3204. Here we will find the value of the address we wish to write to: 3204.

* instruction format :
	
	![instruction format](https://www.andreinc.net/assets/images/2021-12-01-writing-a-simple-vm-in-less-than-125-lines-of-c/sti.drawio.png)

* function format :
	```c
	static inline void sti(uint16_t i){
		mw(mr(reg[RPC] + POFF9(i)), reg[DR(i)]);
	}
	```
### str - Store base + offset
* 使用指定的 Base Address, 並將偏移量(OFFSET6)加入, 而不是 `RPC` 開始
* instruction format :
	
	![](https://www.andreinc.net/assets/images/2021-12-01-writing-a-simple-vm-in-less-than-125-lines-of-c/str.drawio.png)

* function format :
	```c
	static inline void str(uint16_t i){
		mw(reg[SR1(i)] + POFF(i), reg[DR(i)]);
	}
### jmp - Jump
![jmp](https://www.andreinc.net/assets/images/2021-12-01-writing-a-simple-vm-in-less-than-125-lines-of-c/jmpexp.drawio.png)
> Let’s say our RPC=0x3002, and R2=0x3066 (this is BASER). When the jmp instruction is encountered, RPC will jump directly to the memory address kept in BASER (R2=0x3066), and the program flow will continue.

* 通常情況下, `RPC` 會在每條指令執行後自動遞增
* `jmp` 則是將 `RPC` 跳轉到 BASER 內容指定位置的指令
* instruction format :

	![instruction](https://www.andreinc.net/assets/images/2021-12-01-writing-a-simple-vm-in-less-than-125-lines-of-c/jmp.drawio.png)

* get BASER macro :
	```c
	#define BR(i) (((i)>>6)&0x7)
	```
* function format :
	```c
	static inline void jmp(uint16_t i){
		reg[RPC] = reg[BR(i)];
	} 
	```
	>類似於 high-level 程式語言的 `go to`
### jsr - Jump to subroutines
![](https://www.andreinc.net/assets/images/2021-12-01-writing-a-simple-vm-in-less-than-125-lines-of-c/jsrexp.drawio.png)
> In the above example, RPC is initially set to 3002. At this position, it’s a jsr instruction. We store RPC in R7 to remember from where we branched off. Then we jump with the required offset=100, to position 0x3066, and we update RPC to this.

* 實現子程序跳轉
	* 有一個輸入（從暫存器讀取數據）和一個輸出(返回值放入暫存器)
* instruction format : 

	![instruction format](https://www.andreinc.net/assets/images/2021-12-01-writing-a-simple-vm-in-less-than-125-lines-of-c/jsr.drawio.png)

* function format :
	* save the `RPC` in R7(紀錄是從哪裡跳轉的)
	* if bit[11] = 0, set the `RPC` = `BASER`
	* if bit[11] = 1, set the `RPC` = `RPC` + OFFSET11
	```c
	#define FL(i) (((i)>>11)&0x1)
	#define POFF11(i) sext((i)&0x7FF, 11)

	static inline void jsr(uint16_t i){
		reg[R7] = reg[RPC];
		reg[RPC] = (FL(i)) ? reg[RPC] + POFF11(i) : reg[BR(i)];
	}
	```
### br - Conditional branch
* 與 `jsr` 指令相似, 最大的不同在於只有在條件達成的情況下才會跳轉
* instruction format : 
	
	![](https://www.andreinc.net/assets/images/2021-12-01-writing-a-simple-vm-in-less-than-125-lines-of-c/br.drawio.png)
* function format :
	```c
	#define FCND(i) (((i)>>9)&0x7)

	static inline void br(uint16_t i){
		if(reg[RCND] & FCND(i)){
			reg[RPC] += POFF9(i);	
		}
	}
	```
### trap
* 使我們能夠與 I/O 交互, 還能與其他設備交互
* instruction format :
	
	![instruction format](https://www.andreinc.net/assets/images/2021-12-01-writing-a-simple-vm-in-less-than-125-lines-of-c/trap.drawio.png)
	* 使用 `TRAPVECT` 來決定要使用哪個函數
		* 每個可用的 trap 都保存在 `trp_ex` 中, array中包含指向相關C函數的 pointer
* supported 8 trap functions

| Trap Function | TRAPVECT | trp_ex[] index | Comments |
| :-: | :-: | :-: | :-: |
| tgetc | 0x20 | 0 | 從鍵盤讀取字符並複製到 `R0` |
| tout | 0x21 | 1 | 將儲存於 `R0` 字符輸出到控制台 |
| tputs | 0x22 | 2 | 打印字符串。字符保存在連續的內存位置。從 `R0` 中的指定位置開始。如果遇到 0x0000, 則停止打印 |
| tin | 0x23 | 3 | 從鍵盤讀取字符並複製到 `R0` 中, 然後將字符打印至控制台 |
| tputsp | 0x24 | 4 | <mark>未實做。</mark>用於在每個內存位置儲存兩個字符, 而不是一個字符 |
| thalt | 0x25 | 5 | 停止程式的執行。VM 停止運行 |
| tinu16 | 0x26 | 6 | 從鍵盤讀取 uint16_t 並儲存在 `R0` 中 |
| toutu16 | 0x27 | 7 | 將儲存於 `R0` 的 uint16_t 輸出至控制台 |

>[!CAUTION]
>`%hu`: 以 unsigned short 方式 

* function format :
	```c
	#define TRP(i) ((i)&0xFF)

	static inline void tgetc()  { /* code */ }
	static inline void tout()   { /* code */ }
	static inline void tputs()  { /* code */ }
	static inline void tin()    { /* code */ }
	static inline void tputsp() { /* code */ }
	static inline void thalt()  { /* code */ } 
	static inline void tinu16() { /* code */ }
	static inline void toutu16() { /* code */ }

	enum { trp_offset = 0x20 };
	typedef void (*trp_ex_f)();
	trp_ex_f trp_ex[8] = { tgetc, tout, tputs, tin, tputsp, thalt, tinu16, toutu16 };

	static inline void trap(uint16_t i) { 
		trp_ex[TRP(i)-trp_offset](); 
	}
	```

* tgetc
	```c
	static inline void tgetc(){
		reg[R0] = getchar();
	}
	```
* toutc
	```c
	static inline void toutc(){
		fprintf(stdout, "%c", (char)reg[R0]);
	}
	```
* tputs
	```c
	static inline void tputs(){
		uint16_t *p = mem + reg[R0];
		while(*p){
			fprintf(stdout, "%c", (char)*p);
			p++;
		}
	}
	```
* tin
	* 與 tgetc 相似, 唯一不同在儲存至 `R0` 後打印至控制台
	```c
	static inline void tin(){
		reg[R0] = getchar();
		fprintf(stdout, "%c", reg[R0]);
	}
	```
* thalt
	* 為了跟蹤正在運行的虛擬機, 使用全局 bool 變數(running)。一旦調用`thalt`, running 就會被設置為 false, 此時虛擬機就會自動停止
	```c
	static inline void thalt(){
		running =  false;
	}
	```
* tinu16
	* 從鍵盤輸入讀取 `uint16_t` 並儲存至 `R0`
	```c
	static inline void tinu16(){
		fscanf(stdin, "%hu", &reg[R0]);
	}
	```
* toutu16
	* 將 `R0` 的內容以 `uint16_t` 形式寫入至終端(輸出)
	```c
	static inline void toutu16(){
		fprintf(stdout, "%hu\n", reg[R0]);
	}
	```
## Loading and running programs
* the main loop
	```c
	bool running = true;
	uint16_t PC_START = 0x3000;

	void start(uint16_t offset){
		reg[RPC] = PC_START + offset;
		while(running){
			uint16_t i = mr(reg[RPC]++);
			op_ex[OPC(i)](i);
		}
	}
	```
* load programs into VM
	```c
	void ld_img(char *fname, uint16_t offset){
		FILE *in = fopen(fname, "rb");
		if(in == NULL){
			fprintf(stderr, "Can't open file %s.\n", fname);
			exit(1);	
		}
		
		uint16_t *p = mem + PC_START + offset;
		fread(p, sizeof(uint16_t), (UINT16_MAX-PC_START), in);
		fclose(in);
	}
	```
* the main method
	```c
	int main(int argc, char **argv){
		ld_img(argv[1], 0x0);
		start(0x0);
		return 0;
	}
	```
## Test the VM
* 從鍵盤中讀取兩個數字並相加輸出至 `stdout`
* ASM instruction format
	```assembly
	0xF026    //  1111 0000 0010 0110  TRAP tinu16      ;read an uint16_t in R0
	0x1220    //  0001 0010 0010 0000  ADD R1,R0,x0     ;add contents of R0 to R1
	0xF026    //  1111 0000 0010 0110  TRAP tinu16      ;read an uint16_t in R0
	0x1240    //  0001 0010 0010 0000  ADD R1,R1,R0     ;add contents of R0 to R1
	0x1060    //  0001 0000 0110 0000  ADD R0,R1,x0     ;add contents of R1 to R0
	0xF027    //  1111 0000 0010 0111  TRAP toutu16     ;show the contents of R0 to stdout
	0xF025    //  1111 0000 0010 0101  HALT             ;halt
	```
* execute
	```shell
	make
	gcc -Wall --std=c11 sum_program.c
	./vm.out sum.obj
	```