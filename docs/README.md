# Tomasulo Simulator Experiment Report - English version

---
## Design Approach
- Overview
    - Designed the pipeline as required, implementing 3 adders/subtractors, 2 multipliers/dividers, and 2 Load units.
    - Implemented 6 adder/subtractor reservation stations, 3 multiplier/divider reservation stations, and 3 Load Buffers.
    - Implemented 32 floating-point registers, labeled from F0 to F31.
- Hardware
    - See `codes/tomasulo/hardware.py`
    - Implemented the base class `HardWare`, and subclasses `Adder`, `Multer`, `Loader`, and instantiated the required number of units.
- Reservation Stations and Load Buffer
    - See `codes/tomasulo/reservationstation.py`
    - Implemented the base class `RS`, the base class `LoadBuffer`, and the `ReservationStation` class that integrates them.
- Pipeline
    - See `codes/tomasulo/tomasulo.py`
    - Implemented the `Tomasulo` class, where:
        - The `Tomasulo.step(n)` method simulates the pipeline over `n` clock cycles, with `n` defaulting to 1.
            ```python
            while n > 0: # n clock cycles
                self.clock += 1

                # WRITEBACK
                for LB in self.RS.LB.values():
                    ... # Write back the results of instructions that completed in the Load Buffer

                for ARS in self.RS.ARS.values():
                    ... # Write back the results of instructions that completed in the adder/subtractor reservation station

                for MRS in self.RS.MRS.values():
                    ... # Write back the results of instructions that completed in the multiplier/divider reservation station

                # EXECUTE
                for LB in self.RS.LB.values():
                    if LB.busy is True and LB.remain is not None:
                        LB.remain -= 1 # Decrease remaining cycles by 1
                        if LB.remain == 0: # When remaining cycles reach 0, record the instruction completion time
                            if self.inst[LB.inst].ExecComp is None and self.inst[LB.inst].rs == LB.name:
                                self.inst[LB.inst].ExecComp = self.clock
                            Load[LB.FU].free() # Release Load unit resources

                for ARS in self.RS.ARS.values():
                    ... # Similarly

                for MRS in self.RS.MRS.values():
                    ... # Similarly

                # ISSUE
                if self.PC.status is None and self.PC.value < len(self.inst): # If there are instructions to issue
                    inst = self.inst[self.PC.value]
                    res = self.RS.busy(inst)
                    if res is not None:
                        if inst.Issue is None:
                            inst.Issue = self.clock # Record the instruction issue time and the name of the issuing reservation station
                            if inst.op == Config.OP_LD:
                                inst.rs = self.RS.LB[res].name
                            elif inst.op == Config.OP_ADD or inst.op == Config.OP_SUB or inst.op == Config.OP_JUMP:
                                inst.rs = self.RS.ARS[res].name
                            elif inst.op == Config.OP_MUL or inst.op == Config.OP_DIV:
                                inst.rs = self.RS.MRS[res].name
                            else:
                                return

                        if inst.op == Config.OP_LD: # Issue instruction
                            ...
                        elif inst.op == Config.OP_ADD or inst.op == Config.OP_SUB:
                            ...
                        elif inst.op == Config.OP_MUL or inst.op == Config.OP_DIV:
                            ...
                        elif inst.op == Config.OP_JUMP:
                            ...
                        else:
                            return
                        if inst.op != Config.OP_JUMP:
                            self.PC.value += 1 # PC register +1
                    else:
                        pass

                # EXECUTE HardWare
                ready = []
                for LB in self.RS.LB.values():
                    if LB.busy is True and LB.FU is None: # When hardware is ready, fetch the ready instructions from the reservation station
                        ready.append(LB)
                ready = sorted(ready, key=lambda x:x.inst) # Sort by instruction ID
                loaderlist = []
                for loader in Load.values():
                    if loader.status is None:
                        loaderlist.append(loader)
                lenth = min(len(ready), len(loaderlist))
                for i in range(lenth):
                    loader = loaderlist[i] # Send the ready instructions to hardware for execution
                    LB = ready[i]
                    loader.op =  self.inst[LB.inst].op
                    loader.status = LB.name
                    loader.vj = LB.im
                    LB.remain = Config.TIME[loader.op]
                    LB.FU = loader.name

                ready = []
                for ARS in self.RS.ARS.values():
                    ... # Similarly

                ready = []
                for MRS in self.RS.MRS.values():
                    ... # Similarly

                n -= 1
            ```
        - The `Tomasulo.reset()` method resets the pipeline by calling the `free()` method on each unit.
        - The `Tomasulo.insert_inst()` method inputs instructions.

## UI Description
- Interface
    ![ui](./ui.png)
- Buttons
    - Input Instructions
        Enter instructions in the input box or import from a file  
        ![init](./init.png)  
    - Single-Step/Multi-Step Execution
        Debug step by step or enter the number of steps to debug  
    - Auto Run
        Automatically runs the simulator, executing one step per second until completion. During this time, the auto-run button changes to a pause button, and other buttons are disabled  
        ![autorun](./autorun.png)  
    - Run Until Completion
        Runs the simulator until all instructions are executed  
        ![end](./end.png)  
        At this point, only the clear button is available, other buttons are disabled  
        ![result](./result.png)  
    - Clear
        Clears the records and resets the simulator.

## Deployment and Execution
- Compile and Run
    The simulator is written in Python 3.6, and the GUI uses PyQt5. To compile and run, you need to install Python 3.6 and the relevant PyQt5 libraries first. Then, enter the `codes` folder from the command line:  
    - Run the command-line interface
        `python run.py core`  
    - Run the GUI
        `python run.py` or `python run.py gui`  
- Running the Release Version:
    Enter the `release/dist` folder and run `run.exe`.

## Testing
- Test cases are in the `codes/static/test` folder. Among them, `test0.nel` to `test2.nel` are official test cases, and the rest are custom test cases, all of which passed the tests.
- `test3.nel` and `test4.nel` are comprehensive tests, and `test5.nel` is a JUMP test, implementing a loop from 0 to 0x10.
- All test results are correct.
