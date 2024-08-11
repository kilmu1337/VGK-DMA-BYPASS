Thanks to my friend kilmu1337。I will show you how to make your firmware "active"

# What is an active device?
An "active device" is actually a misnomer used by firmware developers. Instead of saying a device is "active," it's more accurate to say that the device frequently initiates DMA (Direct Memory Access) transfers. After a DMA transfer is completed, an interrupt is used to notify the CPU that the data has been successfully transferred. This mechanism is often referred to as the "doorbell" mechanism, which is a very apt analogy—just like a delivery person ringing the doorbell to notify you that your package has arrived, the device "rings the doorbell" to inform the CPU that the data is ready.

In a normal operation, if a device initiates a DMA transfer, there will be an interrupt to notify the CPU. In PCIe (Peripheral Component Interconnect Express), interrupts are typically implemented in three ways: **Legacy Interrupt**, **MSI (Message Signaled Interrupt)**, and **MSI-X (Message Signaled Interrupt Extended)**.

- **Legacy Interrupt** is the traditional method used by PCI devices to send interrupts.
- **MSI** and **MSI-X** are more commonly used in PCIe devices.

Once MSI or MSI-X is enabled in the PCIe device's configuration space (CFG space), the device will use MSI or MSI-X to send interrupts to the CPU.
# How to make your firmware "active"
The way to make your firmware "active" is by using the cfg interrupt interface provided in the Xilinx PCIe IP core to send interrupts to the Root Complex (RC). 
The code example is as follows:
```verilog
    assign ctx.cfg_interrupt_di             = cfg_int_di;
    assign ctx.cfg_pciecap_interrupt_msgnum = cfg_msg_num;
    assign ctx.cfg_interrupt_assert         = cfg_int_assert;
    assign ctx.cfg_interrupt                = cfg_int_valid;
    assign ctx.cfg_interrupt_stat           = cfg_int_stat;
always @ ( posedge clk_pcie ) begin
   if ( rst ) begin
       cfg_int_valid <= 1'b0;
       cfg_msg_num <= 5'b0;
       cfg_int_assert <= 1'b0;
       cfg_int_di <= 8'b0;
       cfg_int_stat <= 1'b0;
   end else if (cfg_int_ready && cfg_int_valid) begin
       cfg_int_valid <= 1'b0;
   end else if (o_int) begin
       cfg_int_valid <= 1'b0;
   end
end
time int_cnt = 0;
always @ ( posedge clk_pcie ) begin
   if (rst) begin
       int_cnt <= 0;
   end else if (int_cnt == 32'd100000) begin
       int_cnt <= 0;
   end else if (int_enable) begin
       int_cnt <= int_cnt + 1;
   end
end

assign o_int = (int_cnt == 32'd100000);
```
This method is widely used, but it has a critical drawback: when the emulated device has an MSI-X capability (MSI-X CAP), this method cannot transmit interrupts. The reason is that the interrupt interface in the Xilinx IP core does not support sending MSI-X type interrupts. Therefore, when the emulated device has an MSI-X CAP, you need to manually construct a TLP (Transaction Layer Packet) to send an interrupt signal to the Root Complex (RC).
According to PCIe documentation, an interrupt is essentially a `MEMWR64` type TLP packet sent to the RC.
The code for manually constructing and sending this packet is as follows:
```verilog
        bit msix_valid;
        bit msix_has_data;
        bit[127:0] msix_tlp;
        assign tlps_static.tdata[127:0] = msix_tlp; 
        assign tlps_static.tkeepdw  = 4'hf;
        assign tlps_static.tlast   = 1'b1;
        assign tlps_static.tuser[0] = 1'b1;
        assign tlps_static.tvalid   = msix_valid;
        assign tlps_static.has_data   = msix_has_data;
        wire      [31:0] HDR_MEMWR64 = 32'b01000000_00000000_00000000_00000001;
        wire       [31:0] MWR64_DW2  = {`_bs16(pcie_id), 8'b00000000, 8'b00001111};
        wire       [31:0] MWR64_DW3  = {i_addr[31:2], 2'b0};
        wire       [31:0] MWR64_DW4  = {i_data};

    always @ ( posedge clk_pcie ) begin
        if ( rst ) begin
            msix_valid <= 1'b0;
            msix_has_data <= 1'b0;
            msix_tlp <= 127'b0;
        end else if (msix_valid) begin
            msix_valid <= 1'b0;
        end else if (msix_has_data && tlps_static.tready) begin
            msix_valid <= 1'b1;
            msix_has_data <= 1'b0;
        end else if (o_int) begin
           // msix_valid <= 1'b1;
            msix_has_data <= 1'b1;
            msix_tlp <= {MWR64_DW4, MWR64_DW3, MWR64_DW2,HDR_MEMWR64};
        end
    end
    time int_cnt = 0;
    always @ ( posedge clk_pcie ) begin
        if (rst) begin
            int_cnt <= 0;
        end else if (int_cnt == 32'd100000) begin
            int_cnt <= 0;
        end else if (int_enable) begin
            int_cnt <= int_cnt + 1;
        end
    end
    assign o_int = (int_cnt == 32'd100000);
```

The complete code will be released on pcileech_pcie_cfg_a7.sv.


# What is "FULL EMU"?
Many firmwares are incorrectly labeled as "FULL EMU" firmware. In reality, most authors are just using BARs (Base Address Registers) obtained by dumping the device after an Arbor scan. Such firmwares are not truly "FULL EMU" firmwares. In the future, I will release an update with a detection method specifically targeting the most widely used Realtek firmware. This method will help distinguish between genuine "FULL EMU" firmware and "DUMP EMU" firmware.

# Credit
    @AceKingSuited (Discord id ace_king_suited/Channel Link: https://discord.gg/E32e7Yca)
    @kilmu1337 (Discord id kilmu1337/Channel Link: https://discord.gg/sXeTPJfpaN)
# Sponsor
    https://scarlet.technology/
