# Memory-Model

# DESIGN UNDER TEST

        //////////////////
        //////////////////
        ///////DUT////////
        //////////////////
        //////////////////

        module memory
        	#(parameter ADDR_WIDTH = 2,
        	parameter DATA_WIDTH = 8
        	)
        	(
        	input clk,
        	input reset,
        
        ///////////////////////////
        ///Control Signals/////////
        ///////////////////////////
        	input [ADDR_WIDTH-1:0] addr,
        	input 		       wr_en,
        	input		       rd_en,
  
        ////////////////////////////
        //////data signal///////////
        ////////////////////////////
        	input [DATA_WIDTH-1:0] wdata,
        	output [DATA_WIDTH-1:0] rdata
        	);
         	reg [DATA_WIDTH-1:0] rdata;
  
        //////////////////////////
        //////memory//////////////
        //////////////////////////
        	reg [DATA_WIDTH-1:0] mem [2**ADDR_WIDTH];
  
        //////////////////
        ////reset/////////
        //////////////////
        	always @(posedge reset)
        	for (int i=0; i<2**ADDR_WIDTH; i++) mem[i]=8'hFF;
  
        //////////////////////////
        ///write data to memory///
        //////////////////////////
        	always @(posedge clk)
        	if(wr_en) mem[addr] <= wdata;
  
        //////////////////////////
        ///Read data from memory///
        //////////////////////////
        	always @(posedge clk)
        	if (rd_en) rdata <= mem[addr];
  
        endmodule
  
  
  
  
# TRANSACTION 

        //////////////////
        //////////////////
        //transaction.sv//
        //////////////////
        //////////////////
        class transaction;

        ///////////////////////////////////
        //declaring the transaction items//
        ///////////////////////////////////
        	rand bit [1:0] addr;
        	rand bit       wr_en;
        	rand bit       rd_en;
        	rand bit [7:0] wdata;
	             bit [7:0] rdata;
	             bit [1:0] cnt;

	      ////////////////////////////////////////////////////////
	      //constraint, to generate any one among write and read//
	      ////////////////////////////////////////////////////////
	      	constraint wr_rd_c { wr_en != rd_en; };
  
	      ////////////////////////////////////////////////////////////////
	      //postrandomize function, displaying randomized value of items//
	      ////////////////////////////////////////////////////////////////
         	function void postrandomize();
				  $display("-------[Trans] post_randomize-------");
	         	if(wr_en) $display("\t addr=%0h\t wr_en=%0h\t wdata=%0h", addr, wr_en, wdata);
	         	if(rd_en) $display("\t addr=%0h\t rd_en=%0h\t", addr,rd_en);
				  $display("-------------------------------------");
          	endfunction 
  
	      //////////////////////////////////
	      ///////deep copy method/////////// 
	      //////////////////////////////////
        	function transaction do_copy();
		         trans.addr  = this.addr;
		         trans.wr_en = this.wr_en;
		         trans.rd_en = this.rd_en;
		         trans.wdata = this.wdata;
		         return trans;
	      	endfunction
      	  endclass
	
	

# GENERATOR

	//////////////////
	//////////////////
	//generator.sv////
	//////////////////
	//////////////////

	class generator;

	///////////////////////////////////
	//declaring the transaction class//
	///////////////////////////////////
	rand transaction trans,tr;

	////////////////////////////////////////////////////
	//repeat count to specify number items to generate//
	////////////////////////////////////////////////////
	int repeat_count;

	/////////////////////////////////////////////////////
	//mailbox to generate and send the packet to driver//
	/////////////////////////////////////////////////////
	mailbox gen2driv;

	///////////////////////////////////////////
	//event to indicate the end of trasaction//
	///////////////////////////////////////////
	event ended;

	//////////////////
	//constructor ////
	//////////////////

	//getting the mailbox handle from env, in order to share the transaction packet between the generator and driver, the same mailbox is shared between both.

	function new (mailbox gen2driv, event ended); 
		this.gen2driv = gen2driv;
		this.ended    = ended;
		trans = new();
	endfunction

	////////
	//Task//
	////////

	//main task, generates(create and randomizes) the repeat_count number of transaction packets and puts into mailbox
	 task main();
		repeat(repeat_count) begin
			trans = new();
			if( !trans.randomize()) $fatal("Gen::trans randomization failed");
			tr = trans.do_copy();
			gen2driv.put(tr);
		end
		->ended; //trigering indicates the end of generation
	endtask
	endclass






#INTERFACE

	////////////////
	////////////////
	//interface.sv//
	////////////////
	////////////////

	interface mem_intf(input logic clk,reset);

	////////////////////////
	//declaring the signal//
	////////////////////////
	logic [1:0] addr;
	logic wr_en;
	logic rd_en;
	logic [7:0] wdata;
	logic [7:0] rdata;

	////////////////////////////////////
	//////driver clocking block/////////
	////////////////////////////////////
	clocking driver_cb @(posedge clk);
		default input #1 output #1;
		output addr;
		output wr_en;
		output rd_en;
		output wdata;
		output rdata;
	endclocking

	////////////////////////////////////
	//////monitor clocking block/////////
	////////////////////////////////////
	clocking monitor_cb @(posedge clk);
		default input #1 output #1;
		output addr;
		output wr_en;
		output rd_en;
		output wdata;
		output rdata;
	endclocking
	endinterface

	////////////////////////////////////
	////////driver modport//////////////
	////////////////////////////////////
	modport DRIVER (clocking driver_cb, input clk, reset);

	////////////////////////////////////
	////////monitor modport/////////////
	////////////////////////////////////
	modport MONITOR (clocking monitor_cb, input clk, reset);

	endinterface

# DRIVER

	//////////////
	//////////////
	//Driver.sv///
	//////////////
	//////////////
	`define DRIV_IF mem_vif.DRIVER.driver_cbclass driver;

	////////////////////////////////////////////
	//used to count the number of transactions//
	////////////////////////////////////////////

	int no_transaction;

	/////////////////////////////////////
	//creating virtual interface handle//
	/////////////////////////////////////

	virtual mem_intf mem_vif;

	///////////////////////////
	//creating mailbox handle//
	///////////////////////////
	mailbox gen2driv;

	///////////////
	//constructor//
	///////////////

	function new(virtual mem_intf mem_vif, mailbox gen2driv);
		this.mem_vif = mem_vif; //getting the interface
		this.gen2driv = gen2driv; //getting the mailbox handles from environment
	endfunction

	/////////////////////////////////////////////////////////////////////
	//Reset task, Reset the Interface signals to default/initial values//
	/////////////////////////////////////////////////////////////////////
	task reset;
	    wait (mem_vif.reset);
		$display("----[DRIVER] ---Reset Started---");
		`DRIV_IF.wr_en <= 0;
		`DRIV_IF.rd_en <= 0;
		`DRIV_IF.addr  <= 0;
		`DRIV_IF.wdata <= 0;
	    wait (!mem_vif.reset);
		$display("---[DRIVER] ---Reset Ended---");
	endtask

	//////////////////////////////////////////////////////
	//drivers the transaction items to interface signals//
	//////////////////////////////////////////////////////
	task drive;
		forever begin
		    transaction trans;
			`DRIV_IF.wr_en <= 0;
			`DRIV_IF.rd_en <= 0;
		    gen2driv.get(trans);
			$display("----[DRIVER-TRANSFER: %0d]---",no_transactions);
		    @(posedge mem_vif.DRIVER.clk);
		end
		
		if(trans.rd_en) begin 
			`DRIV_IF.rd_en <= trans.rd_en;
		    @(posedge mem_vif.DRIVER.clk);
			`DRIV_IF.rd_en <= 0;
		    @(posedge mem_vif.DRIVER.clk);
		    trans.rdata = `DRIV_IF.rdata;
			$display("\tADDR = %0h \tRDATA = %0h",trans.addr, `DRIV_IF.rdata);
		end
			$display("---------------------------------------------");
		no_transactions++;
	endtask
		
	task main;
	     forever begin
		fork
		//Thread-1: Waiting for reset
		begin
		wait(mem_vif.reset);
		end
		//Thread-2: Calling drive taskbegin
		forever
		drive();
		end
		join_any
		disable fork;
	      end
	endtask
	endclass


# MONITOR

	///////////////
	///////////////
	//Monitor.sv///
	///////////////
	///////////////

	//sample the interface signals, capture into transaction packet and send the packet to scoreboard//
	`define MON_IF mem_vif.MONITOR.monitor_cb
	class monitor;

	/////////////////////////////////////
	//creating virtual interface handle//
	/////////////////////////////////////
	virtual mem_intf mem_vif;

	///////////////////////////
	//creating mailbox handle//
	///////////////////////////
	mailbox mon2scb;

	///////////////
	//constructor//
	///////////////
	function new(virtual mem_intf mem_vif, mailbox mon2scb);
		this.mem_vif = mem_vif;  //getting the iterface
		this.mon2scb = mon2scb;  //getting the mailbox handles from environment
	endfunction

	//////////////////////////////////////////////////////////////////
	//Samples the interface signal and send the packet to scoreboard//
	//////////////////////////////////////////////////////////////////
	task main;
		forever begin
		    transaction trans;
		    trans = new();
		
		    @(posedge mem_vif.MONITOR.clk);
		    wait(`MON_IF.rd_en || `MON_IF.wr_en);
		    trans.addr  = `MON_IF.addr;
		    trans.wr_en = `MON_IF.wr_en;
		    trans.wdata = `MON_IF.wdata;
		
		    if (`MON_IF.rd_en) begin
		    trans.rd_en = `MON_IF.rd_en;
		    @(posedge mem_vif.MONITOR.clk);
		    @(posedge mem_vif.MONITOR.clk);
		    trans.rd_en = `MON_IF.rdata;
		end
		
		mon2scb.put(trans);
		
	    end
	  endtask
	endclass




# SCOREBOARD

	//////////////////
	//////////////////
	//Scoreboard.sv///
	//////////////////
	//////////////////
	//get the packetfrom monitor, generated the expected result and compares with the actual result received from monitor
	class scoreboard;

	///////////////////////////
	//creating mailbox handle//
	///////////////////////////
	mailbox mon2scb;

	////////////////////////////////////////////
	//used to count the number of transactions//
	////////////////////////////////////////////
	int no_transaction;

	////////////////////////////////
	//array to use as local memory//
	////////////////////////////////
	bit [7:0] mem[4];

	///////////////
	//constructor//
	///////////////
	function new(mailbox mon2scb);
		this.mon2scb = mon2scb;//getting the mailbox handles from  environment 
		foreach (mem[i] = 8'hFF;
	endfunction

	///////////////////////////////////////////////////////
	//stores wdata and compare rdata with stored data//////
	///////////////////////////////////////////////////////
		task main;
			transaction trans;
			    forever begin
				mon2scb.get(trans);
			if(trans.rd_en) begin
			if(mem[trans.addr] != trans.rdata) 
			$error ("[SCB-FAIL] Addr = %0h, \n \t Data :: Expected = %0h Actual = %0h",trans.addr, mem[trans.addr], trans.rdata);
			else
			$display("[SCB-PASS] Addr = %0h, \n \t Data :: Expected = %0h Actual = %0h",trans.addr, mem[trans.addr], trans,rdata);
			end
			else if(trans.wr_en)
			mem[trans.addr] = trans.wdata;
		
			no_transaction++;
			end
		endtask
	endclass





# ENVIRONMENT

	///////////////////
	///////////////////
	//Environment.sv///
	///////////////////
	///////////////////
	`include "transaction.sv"
	`include "generator.sv"
	`include "driver.sv"
	`include "monitor.sv"
	`include "scoreboard.sv"
	class environment;

	/////////////////////////////////////
	///generator and driver instance/////
	/////////////////////////////////////
	generator gen;
	driver driv;
	monitor mon;
	scoreboard scb;

	///////////////////////////////
	///mailbox handles/////////////
	///////////////////////////////
	mailbox gen2driv;
	mailbox mon2scb;

	///////////////////////////////////////////////////////
	//event for synchronization between generator and test//
	////////////////////////////////////////////////////////
	event gen_ended;

	////////////////////////////////
	////virtual interface///////////
	////////////////////////////////
	virtual mem_intf mem_vif;

	/////////////////////////////////
	////constructor//////////////////
	/////////////////////////////////
	function new(virtual mem_intf mem_vif);
		this.mem_vif = mem_vif;  //get the interface from test
	
		//////////////////////////////////////
		////creating the mailbox /////////////
		//////////////////////////////////////
		gen2driv = new();
		mon2scb = new();

		//////////////////////////////////////
		////creating generator and driver////
		//////////////////////////////////////
		gen  = new(gen2driv, gen_ended);
		driv = new(mem_vif, gen2driv);
		mon  = new(mem_vif, mon2scb);
		scb  = new(mon2scb);

	endfunction

	task pre_test();
		driv.reset();
	endtask

	task test();
		fork
			gen.main();
			driv.main();
			mon.main();
			scb.main();
		join_any
	endtask
	
	task post_test();
		wait (gen_ended.triggered);
		wait(gen.repeat_count == driv.no_transactions)
		wait(gen.repeat_count == scd.no_transactions)
	endtask
	
	////////////////////////////
	///////run task/////////////
	////////////////////////////
		task run;
			pre_test();
			test();
			post_test();
			$finish;
		endtask
	
	endclass


# TESTBENCH

	////////////////
	////////////////
	//testbench.sv//
	////////////////
	////////////////
	//tbench_top or testbench top, this is the top most file, in which DUT(Design Under Test) and verification environment are connected

	////////////////////////
	//including file///// //
	////////////////////////
	//------[NOTE]--------------------//
	//particular testcase can be run by uncomenting, and commenting the rest
	`include "interface.sv"
	//`include "random_test.sv"
	`include "wr_rd_test.sv"
	//`include "default_rd_test.sv"
	//--------------------------------//

	module tbench_top;

	//////////////////////////////////////
	//clock and reset signal declaration//
	//////////////////////////////////////
	bit clk;
	bit reset;

	////////////////////
	//clock generation//
	////////////////////
	always #5 clk= ~clk;

	////////////////////
	//reset generation//
	////////////////////
	initial begin
		reset = 1;
		#5 reset =0;
	end

	//////////////////////////////////
	//creating instance of interface//
	//////////////////////////////////
	intf i_intf(clk,reset);  

	/////////////////////////////////
	//creating instance of testcase//
	/////////////////////////////////
	test t1(i_intf);

	/////////////////////////
	//creating DUT instance//
	/////////////////////////
	memory DUT (
		.clk(intf.clk),
		.reset(intf.reset),
		.addr(intf.addr),
		.wr_en(intf.wr_en),
		.rd_en(intf.rd_en),
		.wdata(intf.wdata),
		.rdata(intf.rdata)
	);

	end
	endmodule



# RANDOM TEST
	
	//////////////////
	//////////////////
	//random_test.sv//
	//////////////////
	//////////////////

	`include "environment.sv"
	program test(mem_intf intf);

	//////////////////////////////////
	//declaring environment instance//
	//////////////////////////////////
	environment env;

	   initial begin
		//creating environment
		env = new(intf);

	//setting the repeat count of generator as 4, means to generate 4 packet
		env.gen.repeat_count = 4;

	//calling run of env, it interns calls generator and driver main tasks
		env.run();
	   end
	endprogram




//////////////////////////
//////////////////////////
//default_rd_test.sv//////
//////////////////////////
//////////////////////////



//////////////////////////
//////////////////////////
///////wr_rd_test.sv//////
//////////////////////////
//////////////////////////







