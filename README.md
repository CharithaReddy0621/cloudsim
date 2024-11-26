# cloudsim
package org.cloudbus.cloudsim.examples;

import java.text.DecimalFormat;
import java.util.ArrayList;
import java.util.Calendar;
import java.util.LinkedList;
import java.util.List;

import org.cloudbus.cloudsim.*;
import org.cloudbus.cloudsim.core.CloudSim;

public class CloudSimExample2 {

    private static List<Cloudlet> cloudletList;
    private static List<Vm> vmlist;

    public static void main(String[] args) {
        Log.printLine("Starting CloudSimExample2...");

        try {
            int num_user = 1; // number of cloud users
            Calendar calendar = Calendar.getInstance();
            boolean trace_flag = false; // disable event tracing

            CloudSim.init(num_user, calendar, trace_flag);

            Datacenter datacenter0 = createDatacenter("Datacenter_0");

            DatacenterBroker broker = createBroker();
            int brokerId = broker.getId();

            vmlist = new ArrayList<>();
            int mipsList[] = {500, 1000, 1500, 2000, 2500, 3000}; // Different MIPS for each VM
            for (int i = 0; i < 6; i++) {
                int vmid = i;
                int mips = mipsList[i];
                long size = 10000; // image size (MB)
                int ram = 512; // vm memory (MB)
                long bw = 1000; // bandwidth
                int pesNumber = 1; // number of CPUs
                String vmm = "Xen";

                Vm vm = new Vm(vmid, brokerId, mips, pesNumber, ram, bw, size, vmm, new CloudletSchedulerTimeShared());
                vmlist.add(vm);
            }
            broker.submitVmList(vmlist);

            cloudletList = new ArrayList<>();
            for (int i = 0; i < 10; i++) {
                int id = i;
                long length = 1000 + (i * 100); // Different lengths
                long fileSize = 300;
                long outputSize = 300;
                UtilizationModel utilizationModel = new UtilizationModelFull();

                Cloudlet cloudlet = new Cloudlet(id, length, 1, fileSize, outputSize, utilizationModel, utilizationModel, utilizationModel);
                cloudlet.setUserId(brokerId);
                cloudlet.setVmId(i % 6); // Assign VMs in a round-robin fashion
                cloudletList.add(cloudlet);
            }
            broker.submitCloudletList(cloudletList);

            // Scenario 1: Space-Shared VM Scheduler
            Log.printLine("\n========== SPACE-SHARED VM SCHEDULER ==========");
            resetSimulation();
            setVmScheduler(datacenter0, new VmSchedulerSpaceShared());
            CloudSim.startSimulation();
            printCloudletList(broker.getCloudletReceivedList());
            CloudSim.stopSimulation();

            // Scenario 2: Time-Shared VM Scheduler
            Log.printLine("\n========== TIME-SHARED VM SCHEDULER ==========");
            resetSimulation();
            setVmScheduler(datacenter0, new VmSchedulerTimeShared());
            CloudSim.startSimulation();
            printCloudletList(broker.getCloudletReceivedList());
            CloudSim.stopSimulation();

            Log.printLine("CloudSimExample2 finished!");
        } catch (Exception e) {
            e.printStackTrace();
            Log.printLine("Unwanted errors happen");
        }
    }

    private static Datacenter createDatacenter(String name) {
        List<Host> hostList = new ArrayList<>();

        int mips = 3000;
        int ram = 8192; // host memory (MB)
        long storage = 1000000; // host storage
        int bw = 10000;

        for (int i = 0; i < 6; i++) {
            List<Pe> peList = new ArrayList<>();
            peList.add(new Pe(0, new PeProvisionerSimple(mips)));
            hostList.add(new Host(i, new RamProvisionerSimple(ram), new BwProvisionerSimple(bw), storage, peList, new VmSchedulerTimeShared(peList)));
        }

        String arch = "x86";
        String os = "Linux";
        String vmm = "Xen";
        double time_zone = 10.0;
        double cost = 3.0;
        double costPerMem = 0.05;
        double costPerStorage = 0.001;
        double costPerBw = 0.0;

        LinkedList<Storage> storageList = new LinkedList<>();
        DatacenterCharacteristics characteristics = new DatacenterCharacteristics(
            arch, os, vmm, hostList, time_zone, cost, costPerMem, costPerStorage, costPerBw
        );

        Datacenter datacenter = null;
        try {
            datacenter = new Datacenter(name, characteristics, new VmAllocationPolicySimple(hostList), storageList, 0);
        } catch (Exception e) {
            e.printStackTrace();
        }

        return datacenter;
    }

    private static DatacenterBroker createBroker() {
        DatacenterBroker broker = null;
        try {
            broker = new DatacenterBroker("Broker");
        } catch (Exception e) {
            e.printStackTrace();
        }
        return broker;
    }

    private static void printCloudletList(List<Cloudlet> list) {
        String indent = "    ";
        Log.printLine();
        Log.printLine("========== OUTPUT ==========");
        Log.printLine("Cloudlet ID" + indent + "STATUS" + indent + "Data center ID" + indent + "VM ID" + indent + "Time" + indent + "Start Time" + indent + "Finish Time");

        DecimalFormat dft = new DecimalFormat("###.##");
        for (Cloudlet cloudlet : list) {
            Log.print(indent + cloudlet.getCloudletId() + indent + indent);

            if (cloudlet.getCloudletStatus() == Cloudlet.SUCCESS) {
                Log.print("SUCCESS");
                Log.printLine(indent + indent + cloudlet.getResourceId() + indent + indent + indent + cloudlet.getVmId() + indent + indent
                        + dft.format(cloudlet.getActualCPUTime()) + indent + indent + dft.format(cloudlet.getExecStartTime()) + indent + indent + dft.format(cloudlet.getFinishTime()));
            }
        }
    }

    private static void setVmScheduler(Datacenter datacenter, VmScheduler scheduler) {
        for (Host host : datacenter.getHostList()) {
            host.setVmScheduler(scheduler);
        }
    }

    private static void resetSimulation() {
        CloudSim.reset();
        cloudletList.clear();
        vmlist.clear();
    }
}
