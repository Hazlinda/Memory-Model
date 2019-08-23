# Memory-Model


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
        input 				 wr_en,
        input					 rd_en,
  
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
        rand bit 		 wr_en;
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
		             transaction trans;
		             trans.addr  = this.addr;
		             trans.wr_en = this.wr_en;
		             trans.rd_en = this.rd_en;
		             trans.wdata = this.wdata;
		             return trans;
	      endfunction
      endclass
	
	



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



//////////////
//////////////
//Driver.sv///
//////////////
//////////////




///////////////
///////////////
//Monitor.sv///
///////////////
///////////////






//////////////////
//////////////////
//Scoreboard.sv///
//////////////////
//////////////////









///////////////////
///////////////////
//Environment.sv///
///////////////////
///////////////////










//////////////
//////////////
///Test.sv////
//////////////
//////////////








/////////////////////
/////////////////////
//Testbench_Top.sv///
/////////////////////
/////////////////////
