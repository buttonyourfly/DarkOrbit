        using System;
using System.Collections.Generic;
using System.ComponentModel;
using System.Data;
using System.Drawing;
using System.Linq;
using System.Text;
using System.Windows.Forms;
using System.Net.Sockets;
using System.Threading;
using System.IO;
using System.Xml.Linq;
using System.Net;

namespace UriWars
{
    public partial class Form1 : Form
    {
        Socket client;
        System.Threading.Timer PingTimer;
        Thread ReceiveThread;
        Thread RedrawThread;
        Thread BotThread;

        MapsParser Maps;

        bool runLogic = true;
        LogicState State = LogicState.Login;

        string UserID = "";
        string SessionID = "";
        string MapName = "";

        /* =============   Collection of Shitz   ============= */

        Classes.Map CurrentMap = null;
        Classes.Hero Hero = null;
        List<Classes.Box> Boxes = new List<Classes.Box>();
        List<Classes.Ore> Ores = new List<Classes.Ore>();
        List<Classes.Portal> Portals = new List<Classes.Portal>();
        List<Classes.Ship> Ships = new List<Classes.Ship>();
        List<Classes.Station> SpaceStations = new List<Classes.Station>();

        /* ============= End Collection of Shitz ============= */

        public Form1(string uid, string sid, string map)
        {
            UserID = uid; SessionID = sid; MapName = map;
            InitializeComponent();
            RedrawThread = new Thread(() =>
            {
                while (true && !this.IsDisposed)
                {
                    try
                    {
                        this.Invoke(new MethodInvoker(this.pictureBox1.Invalidate));
                        Thread.Sleep(100);
                    }
                    catch (ThreadAbortException)
                    {
                        throw;
                    }
                    catch
                    {
                        Thread.Sleep(100);
                    }
                }
            });
            RedrawThread.IsBackground = true;
            RedrawThread.Priority = ThreadPriority.Lowest;
            RedrawThread.Start();
            Maps = new MapsParser();
            Maps.Load("https://fb-uridiumwars.bpsecure.com/src/swf/spacemap/xml/maps.php");
            Connect();
        }

        private void Form1_Load(object sender, EventArgs e)
        {
            System.Diagnostics.Trace.WriteLine("[FORM] Loaded");
        }

        void Connect()
        {
            client = new Socket(AddressFamily.InterNetwork, SocketType.Stream, ProtocolType.Tcp);
            client.Connect(Maps.GetMapIPsByName()[MapName], 8080);
            client.Send(Helpers.CreateLegacyPacket("LOGIN", UserID, SessionID, "2.15", "3"));
            System.Diagnostics.Trace.WriteLine("[LOGIN] " + Helpers.CreateLegacyPacketStr("LOGIN", UserID, SessionID, "2.15", "3"));
            ReceiveThread = new Thread(() =>
                {
                    while (client.Connected)
                    {
                        string pack = Receive();
                        if(pack != null)
                            PacketHandler(pack);
                        Thread.Sleep(1);
                    }
                });
            ReceiveThread.IsBackground = true;
            ReceiveThread.Start();
            client.Send(Helpers.CreateLegacyPacket("RDY", "MAP"));
            Ping();
            BotThread = new Thread(new ThreadStart(RunLogic));
            BotThread.IsBackground = true;
            BotThread.Start();
        }

        void Ping()
        {
            if (PingTimer == null)
            {
                PingTimer = new System.Threading.Timer((e) =>
                {
                    Ping();
                }, null, 25000, 25000);
            }
            client.Send(Helpers.CreateLegacyPacket("PNG"));
        }

        private Byte[] packets;
        private int CurrPos;
        private int Length;
        string Receive()
        {
            int PacketLength = 0;
            int PacketPosition = 0;
            bool Continue = true;
            Byte[] packet = new Byte[5000];
            try
            {
                while (Continue)
                {
                    if (CurrPos == Length)
                    {
                        CurrPos = 0;
                        packets = new Byte[1000];
                        Length = client.Receive(packets, 1000, SocketFlags.None);
                    }
                    if (Length == 0)
                    {
                        return null;
                    }
                    for (; CurrPos < Length; CurrPos++)
                    {
                        if (packets[CurrPos] == 10)
                        {
                            PacketLength = PacketPosition + 1;
                            packet[PacketPosition] = 13;
                            CurrPos++;
                            Continue = false;
                            break;
                        }
                        else if (packets[CurrPos] == 0)
                        {
                            packet[PacketPosition] = 13;
                        }
                        else
                        {
                            packet[PacketPosition] = packets[CurrPos];
                        }
                        PacketPosition++;
                    }
                }
                Byte[] Packet2 = new Byte[PacketLength];
                for (int i = 0; i < PacketLength; i++)
                {
                    Packet2[i] = packet[i];
                }
                return Encoding.UTF8.GetString(Packet2).Replace("\r\r\r", "");
            }
            catch (SocketException)
            {
                return null;
            }
        }

        void PacketHandler(string data)
        {
            System.Diagnostics.Trace.WriteLine(data);
            string[] args = data.Replace("\0\r\n", "").Split('|');
            if (args.Length == 1)
                return;
            switch (args[1])
            {
                case "I":
                    {
                        Hero = new Classes.Hero(int.Parse(args[2]), int.Parse(args[4]), int.Parse(args[27]), args[28], args[3], int.Parse(args[12]), int.Parse(args[13]), int.Parse(args[15]), int.Parse(args[16]), int.Parse(args[29]), int.Parse(args[5]), Convert.ToBoolean(int.Parse(args[30])));
                        break;
                    }
                case "m":
                    {
                        CurrentMap = new Classes.Map(Convert.ToInt32(args[2]), Convert.ToInt32(args[3]), Convert.ToInt32(args[4]));
                        if (CurrentMap.Id == 255)
                        {
                            MessageBox.Show("WARNING YOU ARE IN THE TUTORIAL MAP!");
                            client.Disconnect(true);
                            return;
                        }
                        LoadMap();
                        State = LogicState.Idle;
                        break;
                    }
                case "p":
                    {
                        Portals.Add(new Classes.Portal(Convert.ToInt32(args[2]), Convert.ToInt32(args[5]), Convert.ToInt32(args[6])));
                        break;
                    }
                case "s":
                    {
                        SpaceStations.Add(new Classes.Station(0, Convert.ToInt32(args[7]), Convert.ToInt32(args[8]), Convert.ToInt32(args[5])));
                        break;
                    }

                case "C":
                    {
                        Ships.RemoveAll(ship => ship.UserID.Equals(int.Parse(args[2])));
                        Ships.Add(Classes.Ship.Create(int.Parse(args[2]), int.Parse(args[3]), int.Parse(args[11]), args[5], args[6], int.Parse(args[7]), int.Parse(args[8]), int.Parse(args[9]), int.Parse(args[10]), int.Parse(args[13]), int.Parse(args[14]), Convert.ToBoolean(int.Parse(args[16])), Convert.ToBoolean(int.Parse(args[17]))));
                        break;
                    }

                case "R":
                case "K":
                    {
                        lock (Ships)
                            Ships.RemoveAll(ship => ship.UserID == int.Parse(args[2]));
                        if (Hero.SelectedShip != null && Hero.SelectedShip.UserID == int.Parse(args[2]))
                            Hero.SelectedShip = null;
                        break;
                    }

                case "1":
                    {
                        Classes.Ship shp = Ships.FirstOrDefault(ship => ship.UserID == int.Parse(args[2]));
                        if (shp != null)
                            shp.Move(int.Parse(args[3]), int.Parse(args[4]), int.Parse(args[5]));
                        break;
                    }

                case "A":
                    {
                        switch (args[2])
                        {
                            case "v":
                                {
                                    Hero.SetSpeed(int.Parse(args[3]));
                                    break;
                                }

                            case "HPT":
                                {
                                    Hero.SetHp(int.Parse(args[3]), int.Parse(args[4]));
                                    break;
                                }

                            case "SHD":
                                {
                                    Hero.SetShield(int.Parse(args[3]), int.Parse(args[4]));
                                    break;
                                }
                        }
                        break;
                    }

                case "N":
                    {
                        Classes.Ship shp = Ships.FirstOrDefault(ship => ship.UserID == int.Parse(args[2]));
                        if (shp != null)
                        {
                            shp.SetHp(int.Parse(args[5]), int.Parse(args[6]));
                            shp.SetShield(int.Parse(args[3]), int.Parse(args[4]));
                            Hero.SelectedShip = shp;
                        }
                        break;
                    }

                case "Y":
                    {
                        int id = int.Parse(args[3]);
                        if (Hero.UserID == id)
                        {
                            Hero.UpdateHpShieldNano(int.Parse(args[5]), 0, int.Parse(args[6]));
                        }
                        else
                        {
                            Classes.Ship shp = Ships.FirstOrDefault(ship => ship.UserID == id);
                            if (shp != null)
                            {
                                shp.UpdateHpShieldNano(int.Parse(args[5]), 0, int.Parse(args[6]));
                            }
                        }
                        break;
                    }
                    
                /* =============   Boxes   ============= */
                /* Bonux Box*/
                case "c":
                    {
                        lock (Boxes)
                            if (!AntiBot.honeyBoxes.Contains(args[2]))
                                Boxes.Add(new Classes.Box(args[2], Convert.ToInt32(args[3]), Convert.ToInt32(args[4]), Convert.ToInt32(args[5])));
                            else
                                System.Diagnostics.Trace.WriteLine("[WARNING] Found HoneyBox " + args[2] + "!!", "Warning");
                        break;
                    }
                case "2":
                    {
                        lock (Boxes)
                            Boxes.RemoveAll(p => p.Code.Equals(args[2]));
                        break;
                    }

                /* ORE */
                case "q":
                    {
                        lock (Ores)
                            Ores.RemoveAll(p => p.Code.Equals(args[2]));
                        break;
                    }
                case "r":
                    {
                        lock (Ores)
                            if(!AntiBot.honeyBoxes.Contains(args[2]))
                                Ores.Add(new Classes.Ore(args[2], int.Parse(args[3]), int.Parse(args[4]), int.Parse(args[5])));
                            else
                                System.Diagnostics.Trace.WriteLine("[WARNING] Found HoneyBox " + args[2] + "!!", "Warning");
                        break;
                    }
                /* ============= End Boxes ============= */
            }
        }

        void MoveTo(int x, int y)
        {
            client.Send(Helpers.CreateLegacyPacket("1", x.ToString(), y.ToString(), ((int)Hero.Position.x).ToString(), ((int)Hero.Position.y).ToString()));
            Hero.Move(x, y);
        }

        void MoveTo(double x, double y)
        {
            client.Send(Helpers.CreateLegacyPacket("1", ((int)x).ToString(), ((int)y).ToString(), ((int)Hero.Position.x).ToString(), ((int)Hero.Position.y).ToString()));
            Hero.Move((int)x, (int)y);
        }

        void Repair()
        {
            client.Send(Helpers.CreateLegacyPacket("S", "ROB"));
        }

        void LoadMap()
        {
            if (CurrentMap == null)
                return;

            if (InvokeRequired)
            {
                Invoke(new Action(() => LoadMap()));
                return;
            }

            try
            {
                pictureBox1.BackgroundImageLayout = System.Windows.Forms.ImageLayout.Zoom;
                pictureBox1.BackgroundImage = System.Drawing.Image.FromFile(Environment.CurrentDirectory + @"\images\" + CurrentMap.Id + ".jpg");
            }
            catch (FileNotFoundException)
            {
                try
                {
                    pictureBox1.BackgroundImage = System.Drawing.Image.FromFile(Environment.CurrentDirectory + @"\images\0.jpg");
                }
                catch (FileNotFoundException)
                {
                    Image img = new Bitmap(pictureBox1.Size.Width, pictureBox1.Size.Height);
                    var g = Graphics.FromImage(img);
                    g.FillRectangle(new SolidBrush(Color.Black), 0, 0, pictureBox1.Size.Width, pictureBox1.Size.Height);
                    pictureBox1.BackgroundImage = img;
                }
            }
        }

        void RunLogic()
        {
            double lastDeg = 0;
            while (client.Connected && runLogic)
            {
                if (State == LogicState.Login)
                {
                    Thread.Sleep(100);
                    continue;
                }

                if (State != LogicState.Repairing && State != LogicState.NeedRepair && Hero.Hp <= Hero.Hp * 0.20)
                {
                    State = LogicState.NeedRepair;
                    var port = Portals.OrderBy(p => p.Position.DistanceTo(Hero.Position)).FirstOrDefault();
                    if(port != null)
                    {
                        MoveTo(port.Position.x, port.Position.y);
                        Thread.Sleep(50);
                        continue;
                    }
                }

                if (State == LogicState.NeedRepair)
                {
                    if (!Hero.IsMoving)
                    {
                        State = LogicState.Repairing;
                        Repair();
                        continue;
                    }
                    else
                    {
                        Thread.Sleep(50);
                        continue;
                    }
                }

                if (State == LogicState.Repairing)
                {
                    if (Hero.Hp == Hero.Maxhp)
                    {
                        State = LogicState.Idle;
                        continue;
                    }
                    else
                    {
                        Thread.Sleep(100);
                        continue;
                    }
                }

                if (State == LogicState.SearchingBox)
                {
                    Classes.Box nearestBox = Boxes.OrderBy(box => box.Position.DistanceTo(Hero.Position)).FirstOrDefault();
                    if (nearestBox == null)
                    {
                        if (Hero.IsMoving)
                            Thread.Sleep(50);
                        else
                            MoveTo(Helpers.Random(100, CurrentMap.SizeX), Helpers.Random(100, CurrentMap.SizeY));
                        State = LogicState.Idle;
                    }
                    else
                    {
                        MoveTo(nearestBox.Position.x, nearestBox.Position.y);
                        State = LogicState.CollectingBox;
                        Thread.Sleep(50);
                        continue;
                    }
                }

                if (State == LogicState.CollectingBox)
                {
                    if (Hero.IsMoving)
                    {
                        Thread.Sleep(50);
                        continue;
                    }
                    else
                    {
                        Classes.Box nearestBox = Boxes.OrderBy(box => box.Position.DistanceTo(Hero.Position)).FirstOrDefault();
                        if (nearestBox != null)
                        {
                            if (nearestBox.Position.DistanceTo(Hero.Position) < 50)
                            {
                                client.Send(Helpers.CreateLegacyPacket("x", nearestBox.Code));
                                lock(Boxes)
                                    Boxes.RemoveAll(b => b.Code == nearestBox.Code);
                            }
                            else
                            {
                                MoveTo(nearestBox.Position.x, nearestBox.Position.y);
                            }
                            continue;
                        }
                        else
                        {
                            State = LogicState.Idle;
                            continue;
                        }
                    }
                }

                if (State == LogicState.SearchingAlien)
                {
                    Classes.Ship nearestShip = Ships.OrderBy(ship => ship.Position.DistanceTo(Hero.Position)).FirstOrDefault(ship => ship.IsNPC);
                    if (nearestShip == null)
                    {
                        if (Hero.IsMoving)
                            Thread.Sleep(50);
                        else
                            MoveTo(Helpers.Random(100, CurrentMap.SizeX), Helpers.Random(100, CurrentMap.SizeY));
                    }
                    else
                    {
                        Hero.SelectedShip = nearestShip;
                        MoveTo(nearestShip.Position.x, nearestShip.Position.y);
                        client.Send(Helpers.CreateLegacyPacket("SEL", nearestShip.UserID.ToString()));
                        client.Send(Helpers.CreateLegacyPacket("u", "1"));
                        client.Send(Helpers.CreateLegacyPacket("a", nearestShip.UserID.ToString()));
                        State = LogicState.KillingAlien;
                        Thread.Sleep(50);
                        continue;
                    }
                }

                if (State == LogicState.KillingAlien)
                {
                    Classes.Ship target = Hero.SelectedShip;
                    if (target != null)
                    {
                        /*  deltaY = P2_y - P1_y
                            deltaX = P2_x - P1_x    */
                        double deltaX = target.Position.x - Hero.Position.x;
                        double deltaY = target.Position.y - Hero.Position.y;
                        //double currentDeg = Math.Atan2(deltaY, deltaX) / Math.PI * 180 + 180;
                        Classes.Vector newPosition = new Classes.Vector(target.Position.x + 700 * Math.Cos(lastDeg += 25), target.Position.y + 700 * Math.Sin(lastDeg += 25));
                        MoveTo(newPosition.x, newPosition.y);
                        Thread.Sleep(300);
                        continue;
                    }
                    else
                    {
                        State = LogicState.Idle;
                    }
                }

                if (State == LogicState.Idle)
                {
                    if (Ships.Where(ship => ship.IsNPC).Count() > 0 && Boxes.Count > 0)
                    {
                        State = LogicState.SearchingBox;
                    }
                    else if (Ships.Where(ship => ship.IsNPC).Count() > 0)
                    {
                        State = LogicState.SearchingAlien;
                    }
                    else
                    {
                        State = LogicState.SearchingBox;
                    }
                }
            }
        }

        private void button1_Click(object sender, EventArgs e)
        {
            Connect();
        }

        private void pictureBox1_Paint(object sender, PaintEventArgs e)
        {
            try
            {
                Graphics g = e.Graphics;

                Classes.Station[] stations;
                Classes.Ship[] ships;
                Classes.Box[] boxes;
                Classes.Ore[] ores;
                Classes.Portal[] portals;

                lock (SpaceStations)
                {
                    stations = SpaceStations.ToArray();
                }
                lock (Ships)
                {
                    ships = Ships.ToArray();
                }
                lock (Boxes)
                {
                    boxes = Boxes.ToArray();
                }
                lock (Ores)
                {
                    ores = Ores.ToArray();
                }
                lock (Portals)
                {
                    portals = Portals.ToArray();
                }

                foreach (Classes.Station station in stations)
                {
                    Image img;
                    switch (station.Company)
                    {
                        //case 1:
                        //    img = SH_IT.Properties.Resources.mmo_station;
                        //    break;
                        //case 2:
                        //    img = SH_IT.Properties.Resources.eic_station;
                        //    break;
                        //case 3:
                        //    img = SH_IT.Properties.Resources.vru_station;
                        //    break;
                        default:
                            img = new Bitmap(50, 50);
                            break;
                    }
                    g.DrawImage(img, (float)((station.Position.x - 750) / CurrentMap.ByX), (float)((station.Position.y - 750) / CurrentMap.ByY), 35, 35);
                }

                foreach (Classes.Box box in boxes)
                {
                    Color box_color = Color.Aqua;
                    if (box.Type == 0)
                        continue;
                    else if (box.Type == 1)
                    {
                        box_color = Color.DarkOrange;
                    }
                    else if (box.Type == 2)
                    {
                        box_color = Color.Yellow;
                    }
                    else if (box.Type == 21)
                    {
                        box_color = Color.DarkCyan;
                    }
                    else if (box.Type == 22)
                    {
                        box_color = Color.DarkGoldenrod;
                    }
                    else if (box.Type == 31)
                    {
                        box_color = Color.Magenta;
                    }
                    else if (box.Type == 32)
                    {
                        box_color = Color.Firebrick;
                    }


                    g.FillRectangle(new SolidBrush(box_color), (float)(box.Position.x / CurrentMap.ByX), (float)(box.Position.y / CurrentMap.ByY), 2, 2);
                }

                foreach (Classes.Ore ore in ores)
                {
                    g.FillRectangle(new SolidBrush(Color.Gray), (float)(ore.Position.x / CurrentMap.ByX), (float)(ore.Position.y / CurrentMap.ByY), 2, 2);
                }

                Pen EnemyDot = new Pen(Color.Orange, 1);
                EnemyDot.DashStyle = System.Drawing.Drawing2D.DashStyle.Dash;
                foreach (Classes.Ship ship in ships)
                {
                    Color color = Color.FromArgb(0, 127, 255);

                    if (Hero != null)
                    {
                        if (ship.IsNPC || ship.Faction != Hero.Faction)
                            color = Color.Red;
                        if (ship.ClanDiplomacy == 1)
                            color = Color.Lime;
                        if (ship.ClanDiplomacy == 2)
                            color = Color.DarkOrange;
                        if (ship.ClanDiplomacy == 3)
                            color = Color.Red;
                    }


                    var currPos = ship.Position;
                    g.FillRectangle(new SolidBrush(color), (float)(currPos.x / CurrentMap.ByX), (float)(currPos.y / CurrentMap.ByY), 2, 2);
                    if (!ship.IsNPC && ship.Faction != Hero.Faction && ship.ClanDiplomacy != 1 && ship.ClanDiplomacy != 2)
                    {
                        g.SmoothingMode = System.Drawing.Drawing2D.SmoothingMode.AntiAlias;
                        g.DrawRectangle(EnemyDot, (float)(currPos.x / CurrentMap.ByX) - 3, (float)(currPos.y / CurrentMap.ByY) - 3, 8, 8);
                        g.SmoothingMode = System.Drawing.Drawing2D.SmoothingMode.Default;
                    }
                }

                if (Hero != null && CurrentMap != null)
                {
                    var currPos = Hero.Position;
                    var pen = new Pen(Color.Gray, 0.5f);
                    g.DrawLine(pen, 0, (float)(currPos.y / CurrentMap.ByY), 415, (float)(currPos.y / CurrentMap.ByY));
                    g.DrawLine(pen, (float)(currPos.x / CurrentMap.ByX), 0, (float)(currPos.x / CurrentMap.ByX), 225);
                    if (Hero.IsMoving)
                        g.DrawLine(pen, (float)(currPos.x / CurrentMap.ByX), (float)(currPos.y / CurrentMap.ByY), (float)(Hero.Destination.x / CurrentMap.ByX), (float)(Hero.Destination.y / CurrentMap.ByY));
                }

                g.SmoothingMode = System.Drawing.Drawing2D.SmoothingMode.AntiAlias;
                foreach (Classes.Portal port in portals)
                {
                    g.DrawEllipse(new Pen(Color.FromArgb(255, 255, 255, 255)), (float)((port.Position.x) / CurrentMap.ByX) - 5, (float)((port.Position.y) / CurrentMap.ByY) - 5, 10, 10);
                }

                if (Hero != null && Hero.SelectedShip != null)
                {
                    string HPText = String.Format("HP: {0}/{1}", Hero.SelectedShip.Hp, Hero.SelectedShip.Maxhp);
                    string ShieldText = String.Format("Shield: {0}/{1}", Hero.SelectedShip.Shield, Hero.SelectedShip.Maxshield);
                    string AlienText = Hero.SelectedShip.Username;
                    SizeF HPlength = g.MeasureString(HPText, new Font("Arial", 7, FontStyle.Regular));
                    SizeF Shieldlength = g.MeasureString(ShieldText, new Font("Arial", 7, FontStyle.Regular));
                    SizeF Alienlength = g.MeasureString(AlienText, new Font("Arial", 7, FontStyle.Regular));

                    float width = Math.Max(HPlength.Width, Math.Max(Alienlength.Width, Shieldlength.Width));
                    g.DrawString(ShieldText, new Font("Arial", 7, FontStyle.Regular), new SolidBrush(Color.LightBlue), pictureBox1.Size.Width - width - 2, 212);
                    g.DrawString(HPText, new Font("Arial", 7, FontStyle.Regular), new SolidBrush(Color.LightGreen), pictureBox1.Size.Width - width - 2, 212 - HPlength.Height - 1);
                    g.DrawString(AlienText, new Font("Arial", 7, FontStyle.Regular), new SolidBrush(Color.Red), pictureBox1.Size.Width - width - 2, 212 - HPlength.Height - Shieldlength.Height - 2);
                }
                else
                {
                    g.DrawString("By -jD- and Freakazoide", new Font("Arial", 7, FontStyle.Regular), new SolidBrush(Color.Violet), 308, 210);
                }
            }
            catch
            {
            }
        }

        private void pictureBox1_MouseDown(object sender, MouseEventArgs e)
        {
            int x = (int)Math.Round(e.X * CurrentMap.ByX);
            int y = (int)Math.Round(e.Y * CurrentMap.ByY);
            client.Send(Helpers.CreateLegacyPacket("1", x.ToString(), y.ToString(), ((int)Hero.Position.x).ToString(), ((int)Hero.Position.y).ToString()));
            Hero.Move(x, y);
        }

        enum LogicState
        {
            Login,
            Roaming,
            CollectingBox,
            SearchingBox,
            Fleeing,
            SearchingAlien,
            KillingAlien,
            MovingToPortal,
            Jumping,
            Repairing,
            NeedRepair,
            NeedSell,
            Selling,
            Idle
        }
    }

    static class Helpers
    {
        static Random numGen = new Random();
        public static byte[] CreateLegacyPacket(string content)
        {
            return Encoding.UTF8.GetBytes(content + "\n\0");
        }

        public static byte[] CreateLegacyPacket(params string[] packet)
        {
            string content = String.Join("|", packet) + "\n\0";
            return Encoding.UTF8.GetBytes(content);
        }

        public static string CreateLegacyPacketStr(params string[] packet)
        {
            return String.Join("|", packet) + "\n\0";
        }

        public static int Random(int low, int high)
        {
            return numGen.Next(low, high);
        }
    }

    internal class MapsParser
    {
        WebClient webclient;
        XDocument mapsfile;
        bool IsLoaded;
        string url;

        public MapsParser()
        {
            IsLoaded = false;
            webclient = new WebClient();
            webclient.Proxy = null;
        }

        public void Load(string url)
        {
            this.url = url;
            IsLoaded = false;
            mapsfile = XDocument.Parse(webclient.DownloadString(url));
            IsLoaded = true;
        }

        public Dictionary<int, string> GetMapIPs()
        {
            if (!IsLoaded)
            {
                Load(url);
            }

            Dictionary<int, string> ret_dic = new Dictionary<int, string>();
            XElement root = mapsfile.Element("maps");
            foreach (XElement element in root.Elements("map"))
            {
                ret_dic.Add(Convert.ToInt32(element.Attribute("id").Value), element.Element("gameserverIP").Value);
            }
            return ret_dic;
        }

        public Dictionary<string, string> GetMapIPsByName()
        {
            if (!IsLoaded)
            {
                Load(url);
            }

            Dictionary<string, string> ret_dic = new Dictionary<string, string>();
            XElement root = mapsfile.Element("maps");
            foreach (XElement element in root.Elements("map"))
            {
                if(!ret_dic.ContainsKey(element.Attribute("name").Value))
                    ret_dic.Add(element.Attribute("name").Value, element.Element("gameserverIP").Value);
            }
            return ret_dic;
        }

        public string GetFilteredIPs()
        {
            if (!IsLoaded)
            {
                Load(url);
            }

            XDocument temp = new XDocument(mapsfile);
            XElement root = temp.Element("maps");
            foreach (XElement element in root.Elements("map"))
            {
                element.Element("gameserverIP").Value = "127.0.0.1";
            }
            return temp.ToString();
        }

        public string GetUnFilteredIPs()
        {
            if (!IsLoaded)
            {
                Load(url);
            }

            return mapsfile.ToString();
        }
    }

    static class Classes
    {
        public class Vector
        {
            public Vector()
            {
            }

            public Vector(double x, double y)
            {
                this.x = x;
                this.y = y;
            }
            public double x;
            public double y;

            public double DistanceTo(Vector otherdude)
            {
                return Math.Sqrt(((otherdude.x - x) * (otherdude.x - x)) + ((otherdude.y - y) * (otherdude.y - y)));
            }
        }
        public class Map : ICloneable
        {
            public Map(int id, int sizeX, int sizeY, string name = "")
            {
                Id = id;
                SizeX = sizeX;
                SizeY = sizeY;
                ByX = SizeX / 415;
                ByY = SizeY / 225;
                MapName = name;
            }

            public Map(string name)
            {
                MapName = name;
                Id = 0;
                SizeX = 0;
                SizeY = 0;
                ByX = 0;
                ByY = 0;
            }

            public int Id { get; private set; }
            public int SizeX { get; private set; }
            public int SizeY { get; private set; }
            public float ByX { get; private set; }
            public float ByY { get; private set; }
            public string MapName { get; private set; }

            public void Clear()
            {

            }

            public object Clone()
            {
                Map Copy = new Map(this.MapName);
                Copy.Id = Id;
                Copy.SizeX = SizeX;
                Copy.SizeY = SizeY;
                Copy.ByX = ByX;
                Copy.ByY = ByY;
                return Copy;
            }
        }
        public class Box
        {
            protected string code;
            protected int type;
            protected int x;
            protected int y;

            public Box(string code, int type, int x, int y)
            {
                this.code = code;
                this.type = type;
                this.x = x;
                this.y = y;
            }

            public string Code
            {
                get
                {
                    return code;
                }
            }

            public Vector Position
            {
                get
                {
                    return new Vector(x, y);
                }
            }

            public int Type
            {
                get
                {
                    return type;
                }
            }
        }
        public class Ore
        {
            protected string code;
            protected int type;
            protected int x;
            protected int y;

            public Ore(string code, int type, int x, int y)
            {
                this.code = code;
                this.type = type;
                this.x = x;
                this.y = y;
            }

            public string Code
            {
                get
                {
                    return code;
                }
            }

            public Vector Position
            {
                get
                {
                    return new Vector(x, y);
                }
            }

            public int Type
            {
                get
                {
                    return type;
                }
            }
        }
        public class Ship
        {
            public static Ship Create(int id, int shipId, int rank, string clan, string username, int posX, int posY, int company, int clanId, int clandiplomacy, int galaxygatesdone, bool isNpc, bool cloaked)
            {
                return new Ship(id, shipId, rank, clan, username, posX, posY, company, clanId, clandiplomacy, galaxygatesdone, isNpc, cloaked);
            }

            public Ship(int id, int shipType, int rank, string clan, string username, int posX, int posY, int faction, int clanId, int clandiplomacy, int galaxygatesdone, bool isNpc, bool cloaked)
            {
                UserID = id;
                ShipType = shipType;
                Clan = clan;
                Username = username;
                PosX = posX;
                PosY = posY;
                Faction = faction;
                ClanID = clanId;
                Rank = rank;
                //IsEnemy = isenemy;
                ClanDiplomacy = clandiplomacy;
                GatesDone = galaxygatesdone;
                IsNPC = isNpc;
                Cloaked = cloaked;
                IsTaggedByOtherPlayer = false;

                lastMove = DateTime.Now;
                moveDuration = 0;
            }

            public int UserID { get; protected set; }
            public int ShipType { get; set; }
            public int Rank { get; set; }
            public string Clan { get; set; }
            public string Username { get; set; }
            public string Map { get; set; }
            protected int PosX { get; set; }
            protected int PosY { get; set; }
            public int Faction { get; set; }
            public int ClanID { get; set; }
            public int ClanDiplomacy { get; set; }
            public int GatesDone { get; set; }
            public bool IsNPC { get; set; }
            public bool Cloaked { get; set; }
            public bool IsTaggedByOtherPlayer { get; set; }
            public int Hp { get; set; }
            public int Maxhp { get; set; }
            public int NanoHull { get; set; }
            public int MaxNanoHull { get; set; }
            public int Shield { get; set; }
            public int Maxshield { get; set; }
            public int Title { get; set; }
            public int Velocity { get; set; }

            DateTime lastMove;
            double moveDuration; // MILLISECONDS
            Vector moveDestination;
            Vector direction;
            //double moveDistance;
            bool Moving = false;

            public double DistanceTo(Vector otherdude)
            {
                return Math.Sqrt(((otherdude.x - PosX) * (otherdude.x - PosX)) + ((otherdude.y - PosY) * (otherdude.y - PosY)));
            }

            public void Move(int x, int y, int duration)
            {
                Vector currPosition = Position;
                this.PosX = (int)currPosition.x;
                this.PosY = (int)currPosition.y;
                Moving = true;
                direction = new Vector(x - PosX, y - PosY);
                moveDestination = new Vector(x, y);
                //moveDistance = Math.Sqrt((x * x) + (y * y));
                moveDuration = duration;// * 1000;
                lastMove = DateTime.Now;
            }

            public bool IsMoving
            {
                get
                {
                    return Moving;
                }
            }

            public Vector Position
            {
                get
                {
                    if (Moving)
                    {
                        double timeElapsed = (DateTime.Now - lastMove).TotalMilliseconds;
                        if (timeElapsed < moveDuration)
                        {
                            return new Vector(PosX + (direction.x * (timeElapsed / moveDuration)), PosY + (direction.y * (timeElapsed / moveDuration)));
                        }
                        else
                        {
                            Moving = false;
                            PosX = (int)(PosX + direction.x);
                            PosY = (int)(PosY + direction.y);
                            return new Vector(PosX, PosY);
                        }
                    }
                    else
                    {
                        return new Vector(PosX, PosY);
                    }
                }
            }

            public Vector Destination
            {
                get
                {
                    return moveDestination;
                }
            }

            public void SetHp(int hp, int maxhp)
            {
                Hp = hp;
                Maxhp = maxhp;
            }

            public void SetShield(int shield, int maxshield)
            {
                Shield = shield;
                Maxshield = maxshield;
            }

            public void SetNanoHull(int hull, int maxHull)
            {
                NanoHull = hull;
                MaxNanoHull = maxHull;
            }

            public void UpdateHpShieldNano(int hp, int nanoHull, int shield)
            {
                Hp = hp;
                NanoHull = nanoHull;
                Shield = shield;
            }

            public void UpdateHpShieldNano(int hp, int maxhp, int nanoHull, int maxnano, int shield, int maxshield)
            {
                Hp = hp;
                NanoHull = nanoHull;
                Shield = shield;
                Maxhp = maxhp;
                Maxshield = maxshield;
                MaxNanoHull = maxnano;
            }

            public void SetTitle(int title_id)
            {
                Title = title_id;
            }

            public void UpdatePosition(int x, int y)
            {
                PosX = x;
                PosY = y;
                Moving = false;
            }
        }
        public class Hero : Ship
        {
            public Hero(int id, int shipType, int rank, string clan, string username, int posX, int posY, int faction, int clanId, int galaxygatesdone, int speed, bool cloaked)
                : base(id, shipType, rank, clan, username, posX, posY, faction, clanId, 0, galaxygatesdone, false, cloaked)
            {
                this.Velocity = speed;
            }

            public static Hero Create(int id, int shipId, int rank, string clan, string username, int posX, int posY, int company, int clanId, int galaxygatesdone, int speed, bool cloaked)
            {
                return new Hero(id, shipId, rank, clan, username, posX, posY, company, clanId, galaxygatesdone, speed, cloaked);
            }

            DateTime lastMove;
            double moveDuration; // MILLISECONDS
            Vector moveDestination;
            Vector direction;
            double moveDistance;
            bool Moving = false;

            public Ship SelectedShip = null;
            public Ship PreviousSelectedShip = null;
            public bool AutolockedShip = false;
            public bool Attacking = false;
            public short CurrentAmmo = 26;

            public int RequestedAbility = -1;

            public void Move(int x, int y)
            {
                Vector currPosition = Position;
                this.PosX = (int)currPosition.x;
                this.PosY = (int)currPosition.y;
                Moving = true;
                direction = new Vector(x - PosX, y - PosY);
                moveDestination = new Vector(x, y);
                moveDistance = Math.Sqrt(((x - PosX) * (x - PosX)) + ((y - PosY) * (y - PosY)));
                moveDuration = (moveDistance / Velocity) * 1000;
                lastMove = DateTime.Now;
            }

            public void SetSpeed(int speed)
            {
                Velocity = speed;
                if (Moving)
                    moveDuration = (Math.Sqrt(((moveDestination.x - PosX) * (moveDestination.x - PosX)) + ((moveDestination.y - PosY) * (moveDestination.y - PosY))) / Velocity) * 1000;
            }

            public new bool IsMoving
            {
                get
                {
                    return Moving;
                }
            }

            public double DestinationDistance
            {
                get
                {
                    return moveDistance;
                }
            }

            public new Vector Destination
            {
                get
                {
                    return moveDestination;
                }
            }

            public new Vector Position
            {
                get
                {
                    if (Moving)
                    {
                        double timeElapsed = (DateTime.Now - lastMove).TotalMilliseconds;
                        if (timeElapsed < moveDuration)
                        {
                            return new Vector(PosX + (direction.x * (timeElapsed / moveDuration)), PosY + (direction.y * (timeElapsed / moveDuration)));
                        }
                        else
                        {
                            Moving = false;
                            PosX = (int)(PosX + direction.x);
                            PosY = (int)(PosY + direction.y);
                            return new Vector(PosX, PosY);
                        }
                    }
                    else
                    {
                        return new Vector(PosX, PosY);
                    }
                }
            }
        }
        public class Portal
        {
            protected int id;
            protected int x;
            protected int y;

            public Portal(int id, int x, int y)
            {
                this.id = id;
                this.x = x;
                this.y = y;
            }

            public int ID
            {
                get
                {
                    return id;
                }
            }

            public Vector Position
            {
                get
                {
                    return new Vector(x, y);
                }
            }
        }
        public class Station
        {
            protected int id;
            protected int x;
            protected int y;
            protected int company;

            public Station(int id, int x, int y, int company)
            {
                this.id = id;
                this.x = x;
                this.y = y;
                this.company = company;
            }

            public int ID
            {
                get
                {
                    return id;
                }
            }

            public int Company
            {
                get
                {
                    return company;
                }
            }

            public Vector Position
            {
                get
                {
                    return new Vector(x, y);
                }
            }
        }
    }
}
