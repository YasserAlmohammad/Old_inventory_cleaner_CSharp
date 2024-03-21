# Old_inventory_cleaner_CSharp
an old C# app for cleaning inventory on an old online Game (Archlord)
this code dates back to 2007 and 2009 for online game client manipulation and in memory information retrieval
the app will search the game memory for certain item ids and remove them from the inventory, reads exclusions from a local text file too... this works only in Archlord, which used to be online in the past.

the primary code file is like this

using System;
using System.Collections.Generic;
using System.ComponentModel;
using System.Data;
using System.Drawing;
using System.Linq;
using System.Text;
using System.Windows.Forms;
using System.Diagnostics;
using System.Runtime.InteropServices;
using System.Threading;

namespace CleanerUI
{
    public partial class Form1 : Form
    {
        public static List<int> keepItemIDs = new List<int>();
        [DllImport("Kernel32.dll")]
        public static extern bool ReadProcessMemory
        (
            IntPtr hProcess,
            IntPtr lpBaseAddress,
            byte[] lpBuffer,
            UInt32 nSize,
            ref UInt32 lpNumberOfBytesRead
        );

        [DllImport("User32.Dll")]
        static extern bool PostMessage(IntPtr hWnd, uint msg, uint wParam, uint lParam);

        [DllImport("user32.dll")]
        static extern byte VkKeyScan(char ch);

        [DllImport("user32.dll")]
        public static extern uint MapVirtualKey(uint uCode, uint uMapType);


        const uint WM_KEYDOWN = 0x100;
        const uint WM_UP = 0x101;
        const uint MAPVK_VK_TO_VSC = 0;
        const uint WM_MBUTTONDOWN = 0x0207;
        const uint WM_MBUTTONUP = 0x0208;
        const uint WM_LBUTTONDOWN = 0x0201;
        const uint WM_LBUTTONUP = 0x0202;
        const uint WM_RBUTTONUP = 0x0205;
        const uint WM_RBUTTONDOWN = 0x0204;

        //I inventory button
        private static byte vk = VkKeyScan('I');
        private static uint scanCode = (MapVirtualKey(vk, MAPVK_VK_TO_VSC) << 16) & 0x00FF0000; //u can ignore the right hand of & here since the bits are cleared already; //lparam here
        static Process targetProcess = null;
        public class Point
        {
            public int x;
            public int y;
            public Point(int x, int y)
            {
                this.x = x;
                this.y = y;
            }
            public Point()
            {
                x = 0;
                y = 0;
            }
            public override String ToString()
            {
                return "(" + x + "," + y + ")";
            }
        }

        //only used by mouse movement buttons
        public static void mouseClick(Point targetPoint, bool leftButton/*false for right button click*/)
        {
            uint wparam = 0;
            uint lparam = 0;
            if (leftButton)
            {
                wparam = 0x00000001;
                lparam = ((uint)targetPoint.y << 16) | (uint)targetPoint.x; //x to the lower word, y to the higher
                PostMessage(targetProcess.MainWindowHandle, WM_LBUTTONDOWN, wparam, lparam);
                PostMessage(targetProcess.MainWindowHandle, WM_LBUTTONUP, wparam, lparam);
            }
            else
            { //right click to pass through targets, so u dont click on mobs by mistake
                wparam = 0x00000002;
                lparam = ((uint)targetPoint.y << 16) | (uint)targetPoint.x; //x to the lower word, y to the higher
                PostMessage(targetProcess.MainWindowHandle, WM_RBUTTONDOWN, wparam, lparam);
                PostMessage(targetProcess.MainWindowHandle, WM_RBUTTONUP, wparam, lparam);
            }


            //   Thread.Sleep(200);
            // return true;
        }
        
        public Form1()
        {
            InitializeComponent();
            txtClientName.Text = readProcessName();
            txtTimeInMinutes.Text = ""+10;
            readItemIDs();
            //read default file name
        }

        public static bool running=false;
        public static int interval = 600000; //10minutes
        public static string readProcessName()
        {
            // Read the file and display it line by line.
            string line = null;
            System.IO.StreamReader file =
                   new System.IO.StreamReader("processName.txt");
            try
            {
                line = file.ReadLine();
            }
            catch (Exception ex)
            {
                System.Console.WriteLine("something went wrong while reading item ids");
            }
            finally
            {
                file.Close();
            }
            return line;
        }

        public static void readItemIDs()
        {
            // Read the file and display it line by line.
            System.IO.StreamReader file =
                   new System.IO.StreamReader("keep_item_ids.txt");
            try
            {
                string line;
                while ((line = file.ReadLine()) != null)
                {
                    keepItemIDs.Add(Int32.Parse(line));
                }
            }
            catch (Exception ex)
            {
                System.Console.WriteLine("something went wrong while reading item ids");
            }
            finally
            {
                file.Close();
            }
        }

  //      public static void cleaningThread(){
            
 //       }

      //  Thread cleanThread;
        private void button1_Click(object sender, EventArgs e)
        {
            try
            {
                interval = (Int32.Parse(txtTimeInMinutes.Text))*60000;

                Process[] processList = Process.GetProcessesByName(txtClientName.Text);
                if (processList.Length == 0)
                {
                    Console.WriteLine("no porcess running with the name" + txtClientName.Text);
                    return;
                }
                targetProcess = processList[0];

            }
            catch (Exception ex)
            {
                MessageBox.Show(" invalid time, try again");
                    return;
            }
            timer.Interval = interval;
            timer.Start();
            running = true;
         //   cleanThread = new Thread(cleaningThread);
        //    cleanThread.Start();
            btnStart.Enabled = false;
            btnStop.Enabled = true;
           
        }

        private void btnStop_Click(object sender, EventArgs e)
        {
         //   cleanThread.Suspend();
            running = false;
            timer.Stop();
            btnStop.Enabled = false;
            btnStart.Enabled = true;
        }

        private void Form1_Load(object sender, EventArgs e)
        {

        }

        private void Form1_FormClosing(object sender, FormClosingEventArgs e)
        {
  
        }

        private void Form1_FormClosed(object sender, FormClosedEventArgs e)
        {
         /*   System.Console.WriteLine(" hey ");
            if (running)
            {
                try
                {
              //      cleanThread.Abort();
                    
                }
                catch (Exception ex)
                {
                    System.Console.WriteLine(" hey ex");
                }
            }
          */
        }

        private void timer_Tick(object sender, EventArgs e)
        {
       //     while (running)
       //     {
        //        Thread.Sleep(interval);
                int cellHeight = 53;
                Point bag = new Point(738, 294); //y is fixed, distance between midpoints is 53
                Point firstCell = new Point(bag.x, 335); //53 offset on x,
                Point trashPoint = new Point(732, 541);
                Point confirmButtonPoint = new Point(476, 437);
                int lagtime = 0;

                byte[] buffer = new byte[4];
                uint bytesRead = 0;

                int address = Convert.ToInt32("0089afc4", 16);

                ReadProcessMemory(targetProcess.Handle, (IntPtr)address, buffer, 4, ref bytesRead);

                address = BitConverter.ToInt32(buffer, 0) + 1184;   //0x4a0 == 1184  (int)
                ReadProcessMemory(targetProcess.Handle, (IntPtr)address, buffer, 4, ref bytesRead);

                address = BitConverter.ToInt32(buffer, 0) + 68;     //0x44 == 68 
                ReadProcessMemory(targetProcess.Handle, (IntPtr)address, buffer, 4, ref bytesRead);

                address = BitConverter.ToInt32(buffer, 0) + 12;     //0xc  == 12(int)
                ReadProcessMemory(targetProcess.Handle, (IntPtr)address, buffer, 4, ref bytesRead);

                //from here on we access the bag slots which are stored in a sequential array with 4 bytes 
                //offset from the previous slot
                int firstSlotInBagAddress = BitConverter.ToInt32(buffer, 0);

                bag.x = bag.x + 2 * cellHeight;
                for (int i = 2; i < 4; i++)
                {
                    Thread.Sleep(500);
                    //press bag i tab
                    mouseClick(bag, true);
                    //  cleanBag(i,firstCell,trash,cellHeight); //clean current bag
                    bag.x = bag.x + cellHeight;

                    ////////////////////////////////////// clean bag i
                    int row = 0;
                    int col = 0;
                    Thread.Sleep(200);
                    for (int c = 0; c < 16; c++)
                    {
                        row = c / 4;
                        col = c - row * 4;

                        ///////////////////// obtain the item id in current cell
                        address = firstSlotInBagAddress + (c + i * 16) * 4;
                        ReadProcessMemory(targetProcess.Handle, (IntPtr)address, buffer, 4, ref bytesRead);

                        address = BitConverter.ToInt32(buffer, 0) + 64;  //0x40 == 64
                        ReadProcessMemory(targetProcess.Handle, (IntPtr)address, buffer, 4, ref bytesRead);

                        address = BitConverter.ToInt32(buffer, 0) + 48;  //0x30 == 48
                        ReadProcessMemory(targetProcess.Handle, (IntPtr)address, buffer, 4, ref bytesRead);

                        int id = BitConverter.ToInt32(buffer, 0);
                        //////////////////////////check current if we should delete it or not
                        if (id == 0/*empty cell*/ || keepItemIDs.Contains(id))
                            continue; //dont delete, check the next cell
                        ////////////////////
                        Point cell = new Point(firstCell.x + cellHeight * col, firstCell.y + cellHeight * row);
                        //      for (int i = 0; i < 16; i++)
                        //      {


                        //delete current cell
                        //simple: button down at cell point, up at trash point, then cick confirm
                        uint wparam = 0;
                        uint lparam = 0;
                        wparam = 0x00000001; //left button
                        lparam = ((uint)cell.y << 16) | (uint)cell.x; //x to the lower word, y to the higher
                        PostMessage(targetProcess.MainWindowHandle, WM_LBUTTONDOWN, wparam, lparam);
                        Thread.Sleep(100);
                        lparam = ((uint)trashPoint.y << 16) | (uint)trashPoint.x; //x to the lower word, y to the higher
                        PostMessage(targetProcess.MainWindowHandle, WM_LBUTTONUP, wparam, lparam);
                        Thread.Sleep(200);
                        //confirm
                        mouseClick(confirmButtonPoint, true);
                        Thread.Sleep(200);
                    }
                }
           // }
        }
    }
}
