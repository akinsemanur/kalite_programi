using System;
using System.Collections.Generic;
using System.ComponentModel;
using System.Data;
using System.Drawing;
using System.Text;
using System.Windows.Forms;
using System.Runtime.InteropServices;

///

/// Inclusion of PEAK PCAN-Basic namespace ///
using Peak.Can.Basic;
using TPCANHandle = System.UInt16;
using TPCANBitrateFD = System.String;
using TPCANTimestampFD = System.UInt64;
using static System.Windows.Forms.VisualStyles.VisualStyleElement.Rebar;


namespace ICDIBasic
{
    public partial class Form1 : Form
    {
        #region Structures
        /// <summary>
        /// Message Status structure used to show CAN Messages
        /// in a ListView
        /// </summary>
        private class MessageStatus
        {
            private TPCANMsgFD m_Msg;
            private TPCANTimestampFD m_TimeStamp;
            private TPCANTimestampFD m_oldTimeStamp;
            private int m_iIndex;
            private int m_Count;
            private bool m_bShowPeriod;
            private bool m_bWasChanged;

            public MessageStatus(TPCANMsgFD canMsg, TPCANTimestampFD canTimestamp, int listIndex)
            {
                m_Msg = canMsg;
                m_TimeStamp = canTimestamp;
                m_oldTimeStamp = canTimestamp;
                m_iIndex = listIndex;
                m_Count = 1;
                m_bShowPeriod = true;
                m_bWasChanged = false;
            }

            public void Update(TPCANMsgFD canMsg, TPCANTimestampFD canTimestamp)
            {
                m_Msg = canMsg;
                m_oldTimeStamp = m_TimeStamp;
                m_TimeStamp = canTimestamp;
                m_bWasChanged = true;
                m_Count += 1;
            }

            public TPCANMsgFD CANMsg
            {
                get { return m_Msg; }
            }

            public TPCANTimestampFD Timestamp
            {
                get { return m_TimeStamp; }
            }

            public int Position
            {
                get { return m_iIndex; }
            }

            public string TypeString
            {
                get { return GetMsgTypeString(); }
            }

            public string IdString
            {
                get { return GetIdString(); }
            }

            public string DataString
            {
                get { return GetDataString(); }
            }

            public int Count
            {
                get { return m_Count; }
            }

            public bool ShowingPeriod
            {
                get { return m_bShowPeriod; }
                set
                {
                    if (m_bShowPeriod ^ value)
                    {
                        m_bShowPeriod = value;
                        m_bWasChanged = true;
                    }
                }
            }

            public bool MarkedAsUpdated
            {
                get { return m_bWasChanged; }
                set { m_bWasChanged = value; }
            }

            public string TimeString
            {
                get { return GetTimeString(); }
            }

            private string GetTimeString()
            {
                double fTime;

                fTime = (m_TimeStamp / 1000.0);
                if (m_bShowPeriod)
                    fTime -= (m_oldTimeStamp / 1000.0);
                return fTime.ToString("F1");
            }

            private string GetDataString()
            {
                string strTemp;

                strTemp = "";

                if ((m_Msg.MSGTYPE & TPCANMessageType.PCAN_MESSAGE_RTR) == TPCANMessageType.PCAN_MESSAGE_RTR)
                    return "Remote Request";
                else
                    for (int i = 0; i < Form1.GetLengthFromDLC(m_Msg.DLC, (m_Msg.MSGTYPE & TPCANMessageType.PCAN_MESSAGE_FD) == 0); i++)
                        strTemp += string.Format("{0:X2} ", m_Msg.DATA[i]);

                return strTemp;
            }

            private string GetIdString()
            {
                // We format the ID of the message and show it
                //
                if ((m_Msg.MSGTYPE & TPCANMessageType.PCAN_MESSAGE_EXTENDED) == TPCANMessageType.PCAN_MESSAGE_EXTENDED)
                    return string.Format("{0:X8}h", m_Msg.ID);
                else
                    return string.Format("{0:X3}h", m_Msg.ID);
            }

            private string GetMsgTypeString()
            {
                string strTemp;
                bool isEcho = (m_Msg.MSGTYPE & TPCANMessageType.PCAN_MESSAGE_ECHO) == TPCANMessageType.PCAN_MESSAGE_ECHO;

                if ((m_Msg.MSGTYPE & TPCANMessageType.PCAN_MESSAGE_STATUS) == TPCANMessageType.PCAN_MESSAGE_STATUS)
                    return "STATUS";

                if ((m_Msg.MSGTYPE & TPCANMessageType.PCAN_MESSAGE_ERRFRAME) == TPCANMessageType.PCAN_MESSAGE_ERRFRAME)
                    return "ERROR";

                if ((m_Msg.MSGTYPE & TPCANMessageType.PCAN_MESSAGE_EXTENDED) == TPCANMessageType.PCAN_MESSAGE_EXTENDED)
                    strTemp = "EXT";
                else
                    strTemp = "STD";

                if ((m_Msg.MSGTYPE & TPCANMessageType.PCAN_MESSAGE_RTR) == TPCANMessageType.PCAN_MESSAGE_RTR)
                    strTemp += isEcho ? "/RTR [ ECHO ]" : "/RTR";
                else
                {
                    if ((int)m_Msg.MSGTYPE > (int)TPCANMessageType.PCAN_MESSAGE_EXTENDED)
                    {
                        if (isEcho)
                            strTemp += " [ ECHO";
                        else
                            strTemp += " [ ";
                        if ((m_Msg.MSGTYPE & TPCANMessageType.PCAN_MESSAGE_FD) == TPCANMessageType.PCAN_MESSAGE_FD)
                            strTemp += " FD";
                        if ((m_Msg.MSGTYPE & TPCANMessageType.PCAN_MESSAGE_BRS) == TPCANMessageType.PCAN_MESSAGE_BRS)
                            strTemp += " BRS";
                        if ((m_Msg.MSGTYPE & TPCANMessageType.PCAN_MESSAGE_ESI) == TPCANMessageType.PCAN_MESSAGE_ESI)
                            strTemp += " ESI";
                        strTemp += " ]";
                    }
                }

                return strTemp;
            }

        }
        #endregion

        #region Delegates
        /// <summary>
        /// Read-Delegate Handler
        /// </summary>
        private delegate void ReadDelegateHandler();
        #endregion

        #region Members
        /// <summary>
        /// Saves the desired connection mode
        /// </summary>
        private bool m_IsFD;
        /// <summary>
        /// Saves the handle of a PCAN hardware
        /// </summary>
        private TPCANHandle m_PcanHandle;
        /// <summary>
        /// Saves the baudrate register for a conenction
        /// </summary>
        private TPCANBaudrate m_Baudrate;
        /// <summary>
        /// Saves the type of a non-plug-and-play hardware
        /// </summary>
        private TPCANType m_HwType;
        /// <summary>
        /// Stores the status of received messages for its display
        /// </summary>
        private System.Collections.ArrayList m_LastMsgsList;
        /// <summary>
        /// Read Delegate for calling the function "ReadMessages"
        /// </summary>
        private ReadDelegateHandler m_ReadDelegate;
        /// <summary>
        /// Receive-Event
        /// </summary>
        private System.Threading.AutoResetEvent m_ReceiveEvent;
        /// <summary>
        /// Thread for message reading (using events)
        /// </summary>
        private System.Threading.Thread m_ReadThread;
        /// <summary>
        /// Handles of non plug and play PCAN-Hardware
        /// </summary>
        private TPCANHandle[] m_NonPnPHandles;
        #endregion

        #region Methods
        #region Help functions
        /// <summary>
        /// Convert a CAN DLC value into the actual data length of the CAN/CAN-FD frame.
        /// </summary>
        /// <param name="dlc">A value between 0 and 15 (CAN and FD DLC range)</param>
        /// <param name="isSTD">A value indicating if the msg is a standard CAN (FD Flag not checked)</param>
        /// <returns>The length represented by the DLC</returns>
        public static int GetLengthFromDLC(int dlc, bool isSTD)
        {
            if (dlc <= 8)
                return dlc;

            if (isSTD)
                return 8;

            switch (dlc)
            {
                case 9: return 12;
                case 10: return 16;
                case 11: return 20;
                case 12: return 24;
                case 13: return 32;
                case 14: return 48;
                case 15: return 64;
                default: return dlc;
            }
        }

        /// <summary>
        /// Initialization of PCAN-Basic components
        /// </summary>
        private void InitializeBasicComponents()
        {
            // Creates the list for received messages
            //
            m_LastMsgsList = new System.Collections.ArrayList();
            // Creates the delegate used for message reading
            //
            m_ReadDelegate = new ReadDelegateHandler(ReadMessages);
            // Creates the event used for signalize incomming messages 
            //
            m_ReceiveEvent = new System.Threading.AutoResetEvent(false);
            // Creates an array with all possible non plug-and-play PCAN-Channels
            //
            m_NonPnPHandles = new TPCANHandle[]
            {
            PCANBasic.PCAN_ISABUS1,
            PCANBasic.PCAN_ISABUS2,
            PCANBasic.PCAN_ISABUS3,
            PCANBasic.PCAN_ISABUS4,
            PCANBasic.PCAN_ISABUS5,
            PCANBasic.PCAN_ISABUS6,
            PCANBasic.PCAN_ISABUS7,
            PCANBasic.PCAN_ISABUS8,
            PCANBasic.PCAN_DNGBUS1,
            PCANBasic.PCAN_USBBUS1,
            PCANBasic.PCAN_USBBUS2
          
            
            };

            // Fills and configures the Data of several comboBox components
            //
            FillComboBoxData();

            // Prepares the PCAN-Basic's debug-Log file
            //
            ConfigureLogFile();
        }

        /// <summary>
        /// Configures the Debug-Log file of PCAN-Basic
        /// </summary>
        private void ConfigureLogFile()
        {
            UInt32 iBuffer;

            // Sets the mask to catch all events
            //
            iBuffer = PCANBasic.LOG_FUNCTION_ALL;

            // Configures the log file. 
            // NOTE: The Log capability is to be used with the NONEBUS Handle. Other handle than this will 
            // cause the function fail.
            //
            PCANBasic.SetValue(PCANBasic.PCAN_NONEBUS, TPCANParameter.PCAN_LOG_CONFIGURE, ref iBuffer, sizeof(UInt32));
        }

        /// <summary>
        /// Configures the PCAN-Trace file for a PCAN-Basic Channel
        /// </summary>
        private void ConfigureTraceFile()
        {
            UInt32 iBuffer;
            TPCANStatus stsResult;

            // Configure the maximum size of a trace file to 5 megabytes
            //
            iBuffer = 5;
            stsResult = PCANBasic.SetValue(m_PcanHandle, TPCANParameter.PCAN_TRACE_SIZE, ref iBuffer, sizeof(UInt32));
            if (stsResult != TPCANStatus.PCAN_ERROR_OK)
                IncludeTextMessage(GetFormatedError(stsResult));

            // Configure the way how trace files are created: 
            // * Standard name is used
            // * Existing file is ovewritten, 
            // * Only one file is created.
            // * Recording stopts when the file size reaches 5 megabytes.
            //
            iBuffer = PCANBasic.TRACE_FILE_SINGLE | PCANBasic.TRACE_FILE_OVERWRITE;
            stsResult = PCANBasic.SetValue(m_PcanHandle, TPCANParameter.PCAN_TRACE_CONFIGURE, ref iBuffer, sizeof(UInt32));
            if (stsResult != TPCANStatus.PCAN_ERROR_OK)
                IncludeTextMessage(GetFormatedError(stsResult));
        }

        /// <summary>
        /// Help Function used to get an error as text
        /// </summary>
        /// <param name="error">Error code to be translated</param>
        /// <returns>A text with the translated error</returns>
        private string GetFormatedError(TPCANStatus error)
        {
            StringBuilder strTemp;

            // Creates a buffer big enough for a error-text
            //
            strTemp = new StringBuilder(256);
            // Gets the text using the GetErrorText API function
            // If the function success, the translated error is returned. If it fails,
            // a text describing the current error is returned.
            //
            if (PCANBasic.GetErrorText(error, 0, strTemp) != TPCANStatus.PCAN_ERROR_OK)
                return string.Format("An error occurred. Error-code's text (0x{0:X}) couldn't be retrieved", error);
            else
                return strTemp.ToString();
        }

        /// <summary>
        /// Includes a new line of text into the information Listview
        /// </summary>
        /// <param name="strMsg">Text to be included</param>
        private void IncludeTextMessage(string strMsg)
        {
            lbxInfo.Items.Add(strMsg);
            lbxInfo.SelectedIndex = lbxInfo.Items.Count - 1;
        }

        /// <summary>
        /// Gets the current status of the PCAN-Basic message filter
        /// </summary>
        /// <param name="status">Buffer to retrieve the filter status</param>
        /// <returns>If calling the function was successfull or not</returns>
        private bool GetFilterStatus(out uint status)
        {
            TPCANStatus stsResult;

            // Tries to get the sttaus of the filter for the current connected hardware
            //
            stsResult = PCANBasic.GetValue(m_PcanHandle, TPCANParameter.PCAN_MESSAGE_FILTER, out status, sizeof(UInt32));

            // If it fails, a error message is shown
            //
            if (stsResult != TPCANStatus.PCAN_ERROR_OK)
            {
                MessageBox.Show(GetFormatedError(stsResult));
                return false;
            }
            return true;
        }

        /// <summary>
        /// Configures the data of all ComboBox components of the main-form
        /// </summary>
        private void FillComboBoxData()
        {
            // Channels will be check
            //
            

            // FD Bitrate: 
            //      Arbitration: 1 Mbit/sec 
            //      Data: 2 Mbit/sec
            //
            txtBitrate.Text = "f_clock_mhz=20, nom_brp=5, nom_tseg1=2, nom_tseg2=1, nom_sjw=1, data_brp=2, data_tseg1=3, data_tseg2=1, data_sjw=1";

            // Baudrates 
            //
            m_Baudrate = TPCANBaudrate.PCAN_BAUD_500K;
            // Hardware Type for no plugAndplay hardware
            //
            m_HwType = TPCANType.PCAN_TYPE_ISA;

            // Interrupt for no plugAndplay hardware
            //


            // IO Port for no plugAndplay hardware
            //


            // Parameters for GetValue and SetValue function calls
            //
            cbbParameter.SelectedIndex = 0;
        }

        /// <summary>
        /// Activates/deaactivates the different controls of the main-form according
        /// with the current connection status
        /// </summary>
        /// <param name="bConnected">Current status. True if connected, false otherwise</param>
        private void SetConnectionStatus(bool bConnected)
        {
            // Buttons
            //
            btnInit.Enabled = !bConnected;

            btnWrite.Enabled = bConnected;
            btnRelease.Enabled = bConnected;


            btnGetVersions.Enabled = bConnected;

            btnStatus.Enabled = bConnected;
            btnReset.Enabled = bConnected;

            // ComboBoxs
            //


            // Check-Buttons
            //

            // Hardware configuration and read mode
            //


            // Display messages in grid
            //
            tmrDisplay.Enabled = bConnected;
        }

        /// <summary>
        /// Gets the formated text for a PCAN-Basic channel handle
        /// </summary>
        /// <param name="handle">PCAN-Basic Handle to format</param>
        /// <param name="isFD">If the channel is FD capable</param>
        /// <returns>The formatted text for a channel</returns>
        private string FormatChannelName(TPCANHandle handle, bool isFD)
        {
            TPCANDevice devDevice;
            byte byChannel;

            // Gets the owner device and channel for a 
            // PCAN-Basic handle
            //
            if (handle < 0x100)
            {
                devDevice = (TPCANDevice)(handle >> 4);
                byChannel = (byte)(handle & 0xF);
            }
            else
            {
                devDevice = (TPCANDevice)(handle >> 8);
                byChannel = (byte)(handle & 0xFF);
            }

            // Constructs the PCAN-Basic Channel name and return it
            //
            if (isFD)
                return string.Format("{0}:FD {1} ({2:X2}h)", devDevice, byChannel, handle);
            else
                return string.Format("{0} {1} ({2:X2}h)", devDevice, byChannel, handle);
        }

        /// <summary>
        /// Gets the formated text for a PCAN-Basic channel handle
        /// </summary>
        /// <param name="handle">PCAN-Basic Handle to format</param>
        /// <returns>The formatted text for a channel</returns>
        private string FormatChannelName(TPCANHandle handle)
        {
            return FormatChannelName(handle, false);
        }
        #endregion

        #region Message-proccessing functions
        /// <summary>
        /// Display CAN messages in the Message-ListView
        /// </summary>

        /// <returns>True if the message must be created, false if it must be modified</returns>
        private void ProcessMessage(TPCANMsg theMsg, TPCANTimestamp itsTimeStamp)
        {
            TPCANMsgFD newMsg;
            TPCANTimestampFD newTimestamp;

            newMsg = new TPCANMsgFD();
            newMsg.DATA = new byte[64];
            newMsg.ID = theMsg.ID;
            newMsg.DLC = theMsg.LEN;
            for (int i = 0; i < ((theMsg.LEN > 8) ? 8 : theMsg.LEN); i++)
                newMsg.DATA[i] = theMsg.DATA[i];
            newMsg.MSGTYPE = theMsg.MSGTYPE;

            newTimestamp = itsTimeStamp.micros + (1000UL * itsTimeStamp.millis) + (0x100_000_000UL * 1000UL * itsTimeStamp.millis_overflow);

        }

        /// <summary>
        /// Thread-Function used for reading PCAN-Basic messages
        /// </summary>
        private void CANReadThreadFunc()
        {
            UInt32 iBuffer;
            TPCANStatus stsResult;

            iBuffer = Convert.ToUInt32(m_ReceiveEvent.SafeWaitHandle.DangerousGetHandle().ToInt32());
            // Sets the handle of the Receive-Event.
            //
            stsResult = PCANBasic.SetValue(m_PcanHandle, TPCANParameter.PCAN_RECEIVE_EVENT, ref iBuffer, sizeof(UInt32));

            if (stsResult != TPCANStatus.PCAN_ERROR_OK)
            {
                MessageBox.Show(GetFormatedError(stsResult), "Error!", MessageBoxButtons.OK, MessageBoxIcon.Error);
                return;
            }

            // While this mode is selected

        }

        /// <summary>
        /// Function for reading messages on FD devices
        /// </summary>
        /// <returns>A TPCANStatus error code</returns>
        private TPCANStatus ReadMessageFD()
        {
            TPCANMsgFD CANMsg;
            TPCANTimestampFD CANTimeStamp;
            TPCANStatus stsResult;

            // We execute the "Read" function of the PCANBasic                
            //
            stsResult = PCANBasic.ReadFD(m_PcanHandle, out CANMsg, out CANTimeStamp);


            return stsResult;
        }

        /// <summary>
        /// Function for reading CAN messages on normal CAN devices
        /// </summary>
        /// <returns>A TPCANStatus error code</returns>
        private TPCANStatus ReadMessage()
        {
            TPCANMsg CANMsg;
            TPCANTimestamp CANTimeStamp;
            TPCANStatus stsResult;

            // We execute the "Read" function of the PCANBasic                
            //
            stsResult = PCANBasic.Read(m_PcanHandle, out CANMsg, out CANTimeStamp);
            if (stsResult != TPCANStatus.PCAN_ERROR_QRCVEMPTY)
                // We process the received message
                //
                ProcessMessage(CANMsg, CANTimeStamp);

            return stsResult;
        }

        /// <summary>
        /// Function for reading PCAN-Basic messages
        /// </summary>
        private void ReadMessages()
        {
            TPCANStatus stsResult;

            // We read at least one time the queue looking for messages.
            // If a message is found, we look again trying to find more.
            // If the queue is empty or an error occurr, we get out from
            // the dowhile statement.
            //			
            do
            {
                stsResult = m_IsFD ? ReadMessageFD() : ReadMessage();
                if (stsResult == TPCANStatus.PCAN_ERROR_ILLOPERATION)
                    break;
            } while (btnRelease.Enabled && (!Convert.ToBoolean(stsResult & TPCANStatus.PCAN_ERROR_QRCVEMPTY)));
        }
        #endregion

        #region Event Handlers
        #region Form event-handlers
        /// <summary>
        /// Consturctor
        /// </summary>
        public Form1()
        {
            // Initializes Form's component
            //
            InitializeComponent();
            // Initializes specific components
            //
            InitializeBasicComponents();
        }

        /// <summary>
        /// Form-Closing Function / Finish function
        /// </summary>
        private void Form1_FormClosing(object sender, FormClosingEventArgs e)
        {
            // Releases the used PCAN-Basic channel
            //
            if (btnRelease.Enabled)
                btnRelease_Click(this, new EventArgs());
        }
        #endregion

        #region ComboBox event-handlers
        private void cbbChannel_SelectedIndexChanged(object sender, EventArgs e)
        {
            bool bNonPnP;
            string strTemp;

            // Get the handle fromt he text being shown
            //





            // Determines if the handle belong to a No Plug&Play hardware 
            //

            bNonPnP = m_PcanHandle <= PCANBasic.PCAN_DNGBUS1;
            // Activates/deactivates configuration controls according with the 
            // kind of hardware
            //

        }




        private void cbbParameter_SelectedIndexChanged(object sender, EventArgs e)
        {
            // Activates/deactivates controls according with the selected 
            // PCAN-Basic parameter 
            //
            rdbParamActive.Enabled = (cbbParameter.SelectedIndex != 0) && (cbbParameter.SelectedIndex != 20);

            nudDeviceId.Enabled = !rdbParamActive.Enabled;
            nudDelay.Enabled = !rdbParamActive.Enabled;
            laDeviceOrDelay.Text = (cbbParameter.SelectedIndex == 20) ? "Delay (μs):" : "Device ID (Hex):";
            nudDelay.Visible = cbbParameter.SelectedIndex == 20;
            nudDeviceId.Visible = !nudDelay.Visible;
        }
        #endregion

        #region Button event-handlers


        private void btnInit_Click_1(object sender, EventArgs e)
        {
            TPCANStatus stsResult = PCANBasic.Initialize(PCANBasic.PCAN_USBBUS1, m_Baudrate);
            
            TPCANHandle m_PcanHandle = PCANBasic.PCAN_USBBUS1;
            TPCANType m_HwType = TPCANType.PCAN_TYPE_ISA;


            ushort interruptNumber = 3;


            stsResult = PCANBasic.Initialize(
                m_PcanHandle,
                TPCANBaudrate.PCAN_BAUD_500K,
                m_HwType,
                0,
                interruptNumber);




            SetConnectionStatus(stsResult == TPCANStatus.PCAN_ERROR_OK);

        }

        private void btnRelease_Click(object sender, EventArgs e)
        {
            
            PCANBasic.Uninitialize(m_PcanHandle);
            tmrRead.Enabled = false;
            if (m_ReadThread != null)
            {
                m_ReadThread.Abort();
                m_ReadThread.Join();
                m_ReadThread = null;
            }

           
            SetConnectionStatus(false);
        }





        private void btnParameterSet_Click(object sender, EventArgs e)
        {
            TPCANStatus stsResult;
            UInt32 iBuffer;
            bool bActivate;

            bActivate = rdbParamActive.Checked;

            // Sets a PCAN-Basic parameter value
            //
            switch (cbbParameter.SelectedIndex)
            {
                // The device identifier of a channel will be set  

                case 0:
                    iBuffer = Convert.ToUInt32(nudDeviceId.Value);
                    stsResult = PCANBasic.SetValue(m_PcanHandle, TPCANParameter.PCAN_DEVICE_ID, ref iBuffer, sizeof(UInt32));
                    if (stsResult == TPCANStatus.PCAN_ERROR_OK)
                        IncludeTextMessage("The desired Device-ID was successfully configured");
                    break;
                // The 5 Volt Power feature of a channel will be set
                // the 5 volt power feature  of a channel will be set 

                case 1:
                    iBuffer = (uint)(bActivate ? PCANBasic.PCAN_PARAMETER_ON : PCANBasic.PCAN_PARAMETER_OFF);
                    stsResult = PCANBasic.SetValue(m_PcanHandle, TPCANParameter.PCAN_5VOLTS_POWER, ref iBuffer, sizeof(UInt32));
                    if (stsResult == TPCANStatus.PCAN_ERROR_OK)
                        IncludeTextMessage(string.Format("The USB/PC-Card 5 power was successfully {0}", bActivate ? "activated" : "deactivated"));
                    break;
                // The feature for automatic reset on BUS-OFF will be set
                //
                case 2:
                    iBuffer = (uint)(bActivate ? PCANBasic.PCAN_PARAMETER_ON : PCANBasic.PCAN_PARAMETER_OFF);
                    stsResult = PCANBasic.SetValue(m_PcanHandle, TPCANParameter.PCAN_BUSOFF_AUTORESET, ref iBuffer, sizeof(UInt32));
                    if (stsResult == TPCANStatus.PCAN_ERROR_OK)
                        IncludeTextMessage(string.Format("The automatic-reset on BUS-OFF was successfully {0}", bActivate ? "activated" : "deactivated"));
                    break;
                // The CAN option "Listen Only" will be set
                //
                case 3:
                    iBuffer = (uint)(bActivate ? PCANBasic.PCAN_PARAMETER_ON : PCANBasic.PCAN_PARAMETER_OFF);
                    stsResult = PCANBasic.SetValue(m_PcanHandle, TPCANParameter.PCAN_LISTEN_ONLY, ref iBuffer, sizeof(UInt32));
                    if (stsResult == TPCANStatus.PCAN_ERROR_OK)
                        IncludeTextMessage(string.Format("The CAN option \"Listen Only\" was successfully {0}", bActivate ? "activated" : "deactivated"));
                    break;
                // The feature for logging debug-information will be set
                //
                case 4:
                    iBuffer = (uint)(bActivate ? PCANBasic.PCAN_PARAMETER_ON : PCANBasic.PCAN_PARAMETER_OFF);
                    stsResult = PCANBasic.SetValue(PCANBasic.PCAN_NONEBUS, TPCANParameter.PCAN_LOG_STATUS, ref iBuffer, sizeof(UInt32));
                    if (stsResult == TPCANStatus.PCAN_ERROR_OK)
                        IncludeTextMessage(string.Format("The feature for logging debug information was successfully {0}", bActivate ? "activated" : "deactivated"));
                    break;
                // The channel option "Receive Status" will be set
                //
                case 5:
                    iBuffer = (uint)(bActivate ? PCANBasic.PCAN_PARAMETER_ON : PCANBasic.PCAN_PARAMETER_OFF);
                    stsResult = PCANBasic.SetValue(m_PcanHandle, TPCANParameter.PCAN_RECEIVE_STATUS, ref iBuffer, sizeof(UInt32));
                    if (stsResult == TPCANStatus.PCAN_ERROR_OK)
                        IncludeTextMessage(string.Format("The channel option \"Receive Status\" was set to {0}", bActivate ? "ON" : "OFF"));
                    break;
                // The feature for tracing will be set
                //
                case 7:
                    iBuffer = (uint)(bActivate ? PCANBasic.PCAN_PARAMETER_ON : PCANBasic.PCAN_PARAMETER_OFF);
                    stsResult = PCANBasic.SetValue(m_PcanHandle, TPCANParameter.PCAN_TRACE_STATUS, ref iBuffer, sizeof(UInt32));
                    if (stsResult == TPCANStatus.PCAN_ERROR_OK)
                        IncludeTextMessage(string.Format("The feature for tracing data was successfully {0}", bActivate ? "activated" : "deactivated"));
                    break;

                // The feature for identifying an USB Channel will be set
                //
                case 8:
                    iBuffer = (uint)(bActivate ? PCANBasic.PCAN_PARAMETER_ON : PCANBasic.PCAN_PARAMETER_OFF);
                    stsResult = PCANBasic.SetValue(m_PcanHandle, TPCANParameter.PCAN_CHANNEL_IDENTIFYING, ref iBuffer, sizeof(UInt32));
                    if (stsResult == TPCANStatus.PCAN_ERROR_OK)
                        IncludeTextMessage(string.Format("The procedure for channel identification was successfully {0}", bActivate ? "activated" : "deactivated"));
                    break;

                // The feature for using an already configured speed will be set
                //
                case 10:
                    iBuffer = (uint)(bActivate ? PCANBasic.PCAN_PARAMETER_ON : PCANBasic.PCAN_PARAMETER_OFF);
                    stsResult = PCANBasic.SetValue(m_PcanHandle, TPCANParameter.PCAN_BITRATE_ADAPTING, ref iBuffer, sizeof(UInt32));
                    if (stsResult == TPCANStatus.PCAN_ERROR_OK)
                        IncludeTextMessage(string.Format("The feature for bit rate adaptation was successfully {0}", bActivate ? "activated" : "deactivated"));
                    break;

                // The option "Allow Status Frames" will be set
                //
                case 17:
                    iBuffer = (uint)(bActivate ? PCANBasic.PCAN_PARAMETER_ON : PCANBasic.PCAN_PARAMETER_OFF);
                    stsResult = PCANBasic.SetValue(m_PcanHandle, TPCANParameter.PCAN_ALLOW_STATUS_FRAMES, ref iBuffer, sizeof(UInt32));
                    if (stsResult == TPCANStatus.PCAN_ERROR_OK)
                        IncludeTextMessage(string.Format("The reception of Status frames was successfully {0}", bActivate ? "enabled" : "disabled"));
                    break;

                // The option "Allow RTR Frames" will be set
                //
                case 18:
                    iBuffer = (uint)(bActivate ? PCANBasic.PCAN_PARAMETER_ON : PCANBasic.PCAN_PARAMETER_OFF);
                    stsResult = PCANBasic.SetValue(m_PcanHandle, TPCANParameter.PCAN_ALLOW_RTR_FRAMES, ref iBuffer, sizeof(UInt32));
                    if (stsResult == TPCANStatus.PCAN_ERROR_OK)
                        IncludeTextMessage(string.Format("The reception of RTR frames was successfully {0}", bActivate ? "enabled" : "disabled"));
                    break;

                // The option "Allow Error Frames" will be set
                //
                case 19:
                    iBuffer = (uint)(bActivate ? PCANBasic.PCAN_PARAMETER_ON : PCANBasic.PCAN_PARAMETER_OFF);
                    stsResult = PCANBasic.SetValue(m_PcanHandle, TPCANParameter.PCAN_ALLOW_ERROR_FRAMES, ref iBuffer, sizeof(UInt32));
                    if (stsResult == TPCANStatus.PCAN_ERROR_OK)
                        IncludeTextMessage(string.Format("The reception of Error frames was successfully {0}", bActivate ? "enabled" : "disabled"));
                    break;

                // The option "Interframes Delay" will be set
                //
                case 20:
                    iBuffer = Convert.ToUInt32(nudDelay.Value);
                    stsResult = PCANBasic.SetValue(m_PcanHandle, TPCANParameter.PCAN_INTERFRAME_DELAY, ref iBuffer, sizeof(UInt32));
                    if (stsResult == TPCANStatus.PCAN_ERROR_OK)
                        IncludeTextMessage("The delay between transmitting frames was successfully set");
                    break;

                // The option "Allow Echo Frames" will be set
                //
                case 21:
                    iBuffer = (uint)(bActivate ? PCANBasic.PCAN_PARAMETER_ON : PCANBasic.PCAN_PARAMETER_OFF);
                    stsResult = PCANBasic.SetValue(m_PcanHandle, TPCANParameter.PCAN_ALLOW_ECHO_FRAMES, ref iBuffer, sizeof(UInt32));
                    if (stsResult == TPCANStatus.PCAN_ERROR_OK)
                        IncludeTextMessage(string.Format("The reception of Echo frames was successfully {0}", bActivate ? "enabled" : "disabled"));
                    break;

                // The option "Hard Reset Status" will be set
                //
                case 22:
                    iBuffer = (uint)(bActivate ? PCANBasic.PCAN_PARAMETER_ON : PCANBasic.PCAN_PARAMETER_OFF);
                    stsResult = PCANBasic.SetValue(m_PcanHandle, TPCANParameter.PCAN_HARD_RESET_STATUS, ref iBuffer, sizeof(UInt32));
                    if (stsResult == TPCANStatus.PCAN_ERROR_OK)
                        IncludeTextMessage(string.Format("The activation of a hard reset within the method PCANBasic.Reset was successfully {0}", bActivate ? "enabled" : "disabled"));
                    break;

                // The current parameter is invalid
                //
                default:
                    stsResult = TPCANStatus.PCAN_ERROR_UNKNOWN;
                    MessageBox.Show("Wrong parameter code.");
                    return;
            }

            // If the function fail, an error message is shown
            //
            if (stsResult != TPCANStatus.PCAN_ERROR_OK)
                MessageBox.Show(GetFormatedError(stsResult));
        }

        private void btnParameterGet_Click(object sender, EventArgs e)
        {
            TPCANStatus stsResult;
            UInt32 iBuffer;
            StringBuilder strBuffer;

            strBuffer = new StringBuilder(255);

            // Gets a PCAN-Basic parameter value
            //
            switch (cbbParameter.SelectedIndex)
            {
                // The device identifier of a channel will be retrieved
                //
                case 0:
                    stsResult = PCANBasic.GetValue(m_PcanHandle, TPCANParameter.PCAN_DEVICE_ID, out iBuffer, sizeof(UInt32));
                    if (stsResult == TPCANStatus.PCAN_ERROR_OK)
                        IncludeTextMessage(string.Format("The configured Device-ID is 0x{0:X}", iBuffer));
                    break;
                // The activation status of the 5 Volt Power feature of a channel will be retrieved
                //
                case 1:
                    stsResult = PCANBasic.GetValue(m_PcanHandle, TPCANParameter.PCAN_5VOLTS_POWER, out iBuffer, sizeof(UInt32));
                    if (stsResult == TPCANStatus.PCAN_ERROR_OK)
                        IncludeTextMessage(string.Format("The 5-Volt Power of the USB/PC-Card is {0}", (iBuffer == PCANBasic.PCAN_PARAMETER_ON) ? "ON" : "OFF"));
                    break;
                // The activation status of the feature for automatic reset on BUS-OFF will be retrieved
                //
                case 2:
                    stsResult = PCANBasic.GetValue(m_PcanHandle, TPCANParameter.PCAN_BUSOFF_AUTORESET, out iBuffer, sizeof(UInt32));
                    if (stsResult == TPCANStatus.PCAN_ERROR_OK)
                        IncludeTextMessage(string.Format("The automatic-reset on BUS-OFF is {0}", (iBuffer == PCANBasic.PCAN_PARAMETER_ON) ? "ON" : "OFF"));
                    break;
                // The activation status of the CAN option "Listen Only" will be retrieved
                //
                case 3:
                    stsResult = PCANBasic.GetValue(m_PcanHandle, TPCANParameter.PCAN_LISTEN_ONLY, out iBuffer, sizeof(UInt32));
                    if (stsResult == TPCANStatus.PCAN_ERROR_OK)
                        IncludeTextMessage(string.Format("The CAN option \"Listen Only\" is {0}", (iBuffer == PCANBasic.PCAN_PARAMETER_ON) ? "ON" : "OFF"));
                    break;
                // The activation status for the feature for logging debug-information will be retrieved
                case 4:
                    stsResult = PCANBasic.GetValue(PCANBasic.PCAN_NONEBUS, TPCANParameter.PCAN_LOG_STATUS, out iBuffer, sizeof(UInt32));
                    if (stsResult == TPCANStatus.PCAN_ERROR_OK)
                        IncludeTextMessage(string.Format("The feature for logging debug information is {0}", (iBuffer == PCANBasic.PCAN_PARAMETER_ON) ? "ON" : "OFF"));
                    break;
                // The activation status of the channel option "Receive Status"  will be retrieved
                //
                case 5:
                    stsResult = PCANBasic.GetValue(m_PcanHandle, TPCANParameter.PCAN_RECEIVE_STATUS, out iBuffer, sizeof(UInt32));
                    if (stsResult == TPCANStatus.PCAN_ERROR_OK)
                        IncludeTextMessage(string.Format("The channel option \"Receive Status\" is {0}", (iBuffer == PCANBasic.PCAN_PARAMETER_ON) ? "ON" : "OFF"));
                    break;
                // The Number of the CAN-Controller used by a PCAN-Channel
                // 
                case 6:
                    stsResult = PCANBasic.GetValue(m_PcanHandle, TPCANParameter.PCAN_CONTROLLER_NUMBER, out iBuffer, sizeof(UInt32));
                    if (stsResult == TPCANStatus.PCAN_ERROR_OK)
                        IncludeTextMessage(string.Format("The CAN Controller number is {0}", iBuffer));
                    break;
                // The activation status for the feature for tracing data will be retrieved
                //
                case 7:
                    stsResult = PCANBasic.GetValue(m_PcanHandle, TPCANParameter.PCAN_TRACE_STATUS, out iBuffer, sizeof(UInt32));
                    if (stsResult == TPCANStatus.PCAN_ERROR_OK)
                        IncludeTextMessage(string.Format("The feature for tracing data is {0}", (iBuffer == PCANBasic.PCAN_PARAMETER_ON) ? "ON" : "OFF"));
                    break;
                // The activation status of the Channel Identifying procedure will be retrieved
                //
                case 8:
                    stsResult = PCANBasic.GetValue(m_PcanHandle, TPCANParameter.PCAN_CHANNEL_IDENTIFYING, out iBuffer, sizeof(UInt32));
                    if (stsResult == TPCANStatus.PCAN_ERROR_OK)
                        IncludeTextMessage(string.Format("The identification procedure of the selected channel is {0}", (iBuffer == PCANBasic.PCAN_PARAMETER_ON) ? "ON" : "OFF"));
                    break;
                // The extra capabilities of a hardware will asked
                //
                case 9:
                    stsResult = PCANBasic.GetValue(m_PcanHandle, TPCANParameter.PCAN_CHANNEL_FEATURES, out iBuffer, sizeof(UInt32));
                    if (stsResult == TPCANStatus.PCAN_ERROR_OK)
                    {
                        IncludeTextMessage(string.Format("The channel {0} Flexible Data-Rate (CAN-FD)", ((iBuffer & PCANBasic.FEATURE_FD_CAPABLE) == PCANBasic.FEATURE_FD_CAPABLE) ? "does support" : "DOESN'T SUPPORT"));
                        IncludeTextMessage(string.Format("The channel {0} an inter-frame delay for sending messages", ((iBuffer & PCANBasic.FEATURE_DELAY_CAPABLE) == PCANBasic.FEATURE_DELAY_CAPABLE) ? "does support" : "DOESN'T SUPPORT"));
                        IncludeTextMessage(string.Format("The channel {0} using I/O pins", ((iBuffer & PCANBasic.FEATURE_IO_CAPABLE) == PCANBasic.FEATURE_IO_CAPABLE) ? "does allow" : "DOESN'T ALLOW"));
                    }
                    break;
                // The status of the speed adapting feature will be retrieved
                //
                case 10:
                    stsResult = PCANBasic.GetValue(m_PcanHandle, TPCANParameter.PCAN_BITRATE_ADAPTING, out iBuffer, sizeof(UInt32));
                    if (stsResult == TPCANStatus.PCAN_ERROR_OK)
                        IncludeTextMessage(string.Format("The feature for bit rate adaptation is {0}", (iBuffer == PCANBasic.PCAN_PARAMETER_ON) ? "ON" : "OFF"));
                    break;
                // The bitrate of the connected channel will be retrieved (BTR0-BTR1 value)
                //
                case 11:
                    stsResult = PCANBasic.GetValue(m_PcanHandle, TPCANParameter.PCAN_BITRATE_INFO, out iBuffer, sizeof(UInt32));
                    if (stsResult == TPCANStatus.PCAN_ERROR_OK)
                        IncludeTextMessage(string.Format("The bit rate of the channel is 0x{0:X4}", iBuffer));
                    break;
                // The bitrate of the connected FD channel will be retrieved (String value)
                //
                case 12:
                    stsResult = PCANBasic.GetValue(m_PcanHandle, TPCANParameter.PCAN_BITRATE_INFO_FD, strBuffer, 255);
                    if (stsResult == TPCANStatus.PCAN_ERROR_OK)
                    {
                        IncludeTextMessage("The bit rate FD of the channel is represented by the following values:");
                        foreach (string strPart in strBuffer.ToString().Split(','))
                            IncludeTextMessage("   * " + strPart);
                    }
                    break;
                // The nominal speed configured on the CAN bus
                //
                case 13:
                    stsResult = PCANBasic.GetValue(m_PcanHandle, TPCANParameter.PCAN_BUSSPEED_NOMINAL, out iBuffer, sizeof(UInt32));
                    if (stsResult == TPCANStatus.PCAN_ERROR_OK)
                        IncludeTextMessage(string.Format("The nominal speed of the channel is {0} bit/s", iBuffer));
                    break;
                // The data speed configured on the CAN bus
                //
                case 14:
                    stsResult = PCANBasic.GetValue(m_PcanHandle, TPCANParameter.PCAN_BUSSPEED_DATA, out iBuffer, sizeof(UInt32));
                    if (stsResult == TPCANStatus.PCAN_ERROR_OK)
                        IncludeTextMessage(string.Format("The data speed of the channel is {0} bit/s", iBuffer));
                    break;
                // The IP address of a LAN channel as string, in IPv4 format
                //
                case 15:
                    stsResult = PCANBasic.GetValue(m_PcanHandle, TPCANParameter.PCAN_IP_ADDRESS, strBuffer, 255);
                    if (stsResult == TPCANStatus.PCAN_ERROR_OK)
                        IncludeTextMessage(string.Format("The IP address of the channel is {0}", strBuffer.ToString()));
                    break;
                // The running status of the LAN Service
                //
                case 16:
                    stsResult = PCANBasic.GetValue(PCANBasic.PCAN_NONEBUS, TPCANParameter.PCAN_LAN_SERVICE_STATUS, out iBuffer, sizeof(UInt32));
                    if (stsResult == TPCANStatus.PCAN_ERROR_OK)
                        IncludeTextMessage(string.Format("The LAN service is {0}", (iBuffer == PCANBasic.SERVICE_STATUS_RUNNING) ? "running" : "NOT running"));
                    break;
                // The reception of Status frames
                //
                case 17:
                    stsResult = PCANBasic.GetValue(m_PcanHandle, TPCANParameter.PCAN_ALLOW_STATUS_FRAMES, out iBuffer, sizeof(UInt32));
                    if (stsResult == TPCANStatus.PCAN_ERROR_OK)
                        IncludeTextMessage(string.Format("The reception of Status frames is {0}", (iBuffer == PCANBasic.PCAN_PARAMETER_ON) ? "enabled" : "disabled"));
                    break;
                // The reception of RTR frames
                //
                case 18:
                    stsResult = PCANBasic.GetValue(m_PcanHandle, TPCANParameter.PCAN_ALLOW_RTR_FRAMES, out iBuffer, sizeof(UInt32));
                    if (stsResult == TPCANStatus.PCAN_ERROR_OK)
                        IncludeTextMessage(string.Format("The reception of RTR frames is {0}", (iBuffer == PCANBasic.PCAN_PARAMETER_ON) ? "enabled" : "disabled"));
                    break;
                // The reception of Error frames
                //
                case 19:
                    stsResult = PCANBasic.GetValue(m_PcanHandle, TPCANParameter.PCAN_ALLOW_ERROR_FRAMES, out iBuffer, sizeof(UInt32));
                    if (stsResult == TPCANStatus.PCAN_ERROR_OK)
                        IncludeTextMessage(string.Format("The reception of Error frames is {0}", (iBuffer == PCANBasic.PCAN_PARAMETER_ON) ? "enabled" : "disabled"));
                    break;
                // The Interframe delay of an USB channel will be retrieved
                //
                case 20:
                    stsResult = PCANBasic.GetValue(m_PcanHandle, TPCANParameter.PCAN_INTERFRAME_DELAY, out iBuffer, sizeof(UInt32));
                    if (stsResult == TPCANStatus.PCAN_ERROR_OK)
                        IncludeTextMessage(string.Format("The configured interframe delay is {0} μs", iBuffer));
                    break;
                // The reception of Echo frames
                //
                case 21:
                    stsResult = PCANBasic.GetValue(m_PcanHandle, TPCANParameter.PCAN_ALLOW_ECHO_FRAMES, out iBuffer, sizeof(UInt32));
                    if (stsResult == TPCANStatus.PCAN_ERROR_OK)
                        IncludeTextMessage(string.Format("The reception of Echo frames is {0}", (iBuffer == PCANBasic.PCAN_PARAMETER_ON) ? "enabled" : "disabled"));
                    break;
                // The activation of Hard Reset
                case 22:
                    stsResult = PCANBasic.GetValue(m_PcanHandle, TPCANParameter.PCAN_HARD_RESET_STATUS, out iBuffer, sizeof(UInt32));
                    if (stsResult == TPCANStatus.PCAN_ERROR_OK)
                        IncludeTextMessage(string.Format("The method PCANBasic.Reset is {0} a hardware reset", (iBuffer == PCANBasic.PCAN_PARAMETER_ON) ? "performing" : "NOT performing"));
                    break;

                // The direction of the communication with a LAN channel
                //
                case 23:
                    stsResult = PCANBasic.GetValue(m_PcanHandle, TPCANParameter.PCAN_LAN_CHANNEL_DIRECTION, out iBuffer, sizeof(UInt32));
                    if (stsResult == TPCANStatus.PCAN_ERROR_OK)
                    {
                        switch (iBuffer)
                        {
                            case PCANBasic.LAN_DIRECTION_READ:
                                IncludeTextMessage("The communication flow is: incoming only");
                                break;
                            case PCANBasic.LAN_DIRECTION_WRITE:
                                IncludeTextMessage("The communication flow is: outgoing only");
                                break;
                            case PCANBasic.LAN_DIRECTION_READ_WRITE:
                                IncludeTextMessage("The communication flow is: bidirectional");
                                break;
                            default:
                                IncludeTextMessage(string.Format("The communication flow is: undefined (0x{0:X4})", iBuffer));
                                break;
                        }
                    }
                    break;

                // The current parameter is invalid
                //
                default:
                    stsResult = TPCANStatus.PCAN_ERROR_UNKNOWN;
                    MessageBox.Show("Wrong parameter code.");
                    return;
            }

            // If the function fail, an error message is shown
            //
            if (stsResult != TPCANStatus.PCAN_ERROR_OK)
                MessageBox.Show(GetFormatedError(stsResult));
        }



        private void btnGetVersions_Click(object sender, EventArgs e)
        {
            TPCANStatus stsResult;
            StringBuilder strTemp;
            string[] strArrayVersion;

            strTemp = new StringBuilder(256);

            // We get the vesion of the PCAN-Basic API
            //
            stsResult = PCANBasic.GetValue(PCANBasic.PCAN_NONEBUS, TPCANParameter.PCAN_API_VERSION, strTemp, 256);
            if (stsResult == TPCANStatus.PCAN_ERROR_OK)
            {
                IncludeTextMessage("API Version: " + strTemp.ToString());

                // We get the version of the firmware on the device
                //
                stsResult = PCANBasic.GetValue(m_PcanHandle, TPCANParameter.PCAN_FIRMWARE_VERSION, strTemp, 256);
                if (stsResult == TPCANStatus.PCAN_ERROR_OK)
                    IncludeTextMessage("Firmare Version: " + strTemp.ToString());

                // We get the driver version of the channel being used
                //
                stsResult = PCANBasic.GetValue(m_PcanHandle, TPCANParameter.PCAN_CHANNEL_VERSION, strTemp, 256);
                if (stsResult == TPCANStatus.PCAN_ERROR_OK)
                {
                    // Because this information contains line control characters (several lines)
                    // we split this also in several entries in the Information List-Box
                    //
                    strArrayVersion = strTemp.ToString().Split(new char[] { '\n' });
                    IncludeTextMessage("Channel/Driver Version: ");
                    for (int i = 0; i < strArrayVersion.Length; i++)
                        IncludeTextMessage("     * " + strArrayVersion[i]);
                }
            }

            // If an error ccurred, a message is shown
            //
            if (stsResult != TPCANStatus.PCAN_ERROR_OK)
                MessageBox.Show(GetFormatedError(stsResult));
        }



        private void btnInfoClear_Click(object sender, EventArgs e)
        {
            // The information contained in the Information List-Box 
            // is cleared
            //
            lbxInfo.Items.Clear();
        }

        private TPCANStatus WriteFrame()
        {
            TPCANMsg CANMsg;
            TextBox txtbCurrentTextBox;

            // We create a TPCANMsg message structure 
            //
            CANMsg = new TPCANMsg();
            CANMsg.DATA = new byte[8];

            // We configurate the Message.  The ID,
            // Length of the Data, Message Type
            // and the data
            //
            CANMsg.ID = Convert.ToUInt32(txtID.Text, 16);

            CANMsg.MSGTYPE = (chbExtended.Checked) ? TPCANMessageType.PCAN_MESSAGE_EXTENDED : TPCANMessageType.PCAN_MESSAGE_STANDARD;
            // If a remote frame will be sent, the data bytes are not important.
            //


            // The message is sent to the configured hardware
            //
            return PCANBasic.Write(m_PcanHandle, ref CANMsg);
        }

        private TPCANStatus WriteFrameFD()
        {
            TPCANMsgFD CANMsg;
            TextBox txtbCurrentTextBox;
            int iLength;

            // We create a TPCANMsgFD message structure 
            //
            CANMsg = new TPCANMsgFD();
            CANMsg.DATA = new byte[64];

            // We configurate the Message.  The ID,
            // Length of the Data, Message Type 
            // and the data
            //
            CANMsg.ID = Convert.ToUInt32(txtID.Text, 16);

            CANMsg.MSGTYPE = (chbExtended.Checked) ? TPCANMessageType.PCAN_MESSAGE_EXTENDED : TPCANMessageType.PCAN_MESSAGE_STANDARD;
            CANMsg.MSGTYPE |= (chbFD.Checked) ? TPCANMessageType.PCAN_MESSAGE_FD : TPCANMessageType.PCAN_MESSAGE_STANDARD;


            // If a remote frame will be sent, the data bytes are not important.
            //


            // The message is sent to the configured hardware
            //
            return PCANBasic.WriteFD(m_PcanHandle, ref CANMsg);
        }

        private void btnWrite_Click(object sender, EventArgs e)
        {
            TPCANStatus stsResult;

            // Send the message
            //
            stsResult = m_IsFD ? WriteFrameFD() : WriteFrame();

            // The message was successfully sent
            //
            if (stsResult == TPCANStatus.PCAN_ERROR_OK)
                IncludeTextMessage("Message was successfully SENT");
            // An error occurred.  We show the error.
            //			
            else
                MessageBox.Show(GetFormatedError(stsResult));
        }

        private void btnReset_Click(object sender, EventArgs e)
        {
            TPCANStatus stsResult;

            // Resets the receive and transmit queues of a PCAN Channel.
            //
            stsResult = PCANBasic.Reset(m_PcanHandle);

            // If it fails, a error message is shown
            //
            if (stsResult != TPCANStatus.PCAN_ERROR_OK)
                MessageBox.Show(GetFormatedError(stsResult));
            else
                IncludeTextMessage("Receive and transmit queues successfully reset");
        }

        private void btnStatus_Click(object sender, EventArgs e)
        {
            TPCANStatus stsResult;
            String errorName;

            // Gets the current BUS status of a PCAN Channel.
            //
            stsResult = PCANBasic.GetStatus(m_PcanHandle);

            // Switch On Error Name
            //
            switch (stsResult)
            {
                case TPCANStatus.PCAN_ERROR_INITIALIZE:
                    errorName = "PCAN_ERROR_INITIALIZE";
                    break;

                case TPCANStatus.PCAN_ERROR_BUSLIGHT:
                    errorName = "PCAN_ERROR_BUSLIGHT";
                    break;

                case TPCANStatus.PCAN_ERROR_BUSHEAVY: // TPCANStatus.PCAN_ERROR_BUSWARNING
                    errorName = m_IsFD ? "PCAN_ERROR_BUSWARNING" : "PCAN_ERROR_BUSHEAVY";
                    break;

                case TPCANStatus.PCAN_ERROR_BUSPASSIVE:
                    errorName = "PCAN_ERROR_BUSPASSIVE";
                    break;

                case TPCANStatus.PCAN_ERROR_BUSOFF:
                    errorName = "PCAN_ERROR_BUSOFF";
                    break;

                case TPCANStatus.PCAN_ERROR_OK:
                    errorName = "PCAN_ERROR_OK";
                    break;

                default:
                    errorName = "See Documentation";
                    break;
            }

            // Display Message
            //
            IncludeTextMessage(String.Format("Status: {0} (0x{1:X}h)", errorName, stsResult));
        }
        #endregion

        #region Timer event-handler
        private void tmrRead_Tick(object sender, EventArgs e)
        {
            // Checks if in the receive-queue are currently messages for read
            // 
            ReadMessages();
        }


        #endregion

        #region Message List-View event-handler

        #endregion

        #region Information List-Box event-handler
        private void lbxInfo_DoubleClick(object sender, EventArgs e)
        {
            // Clears the content of the Information List-Box
            //
            btnInfoClear_Click(this, new EventArgs());
        }
        #endregion

        #region Textbox event handlers
        private void txtID_Leave(object sender, EventArgs e)
        {
            int iTextLength;
            uint uiMaxValue;

            // Calculates the text length and Maximum ID value according
            // with the Message Type
            //
            iTextLength = (chbExtended.Checked) ? 8 : 3;
            uiMaxValue = (chbExtended.Checked) ? (uint)0x1FFFFFFF : (uint)0x7FF;

            // The Textbox for the ID is represented with 3 characters for 
            // Standard and 8 characters for extended messages.
            // Therefore if the Length of the text is smaller than TextLength,  
            // we add "0"
            //
            while (txtID.Text.Length != iTextLength)
                txtID.Text = ("0" + txtID.Text);

            // We check that the ID is not bigger than current maximum value
            //
            if (Convert.ToUInt32(txtID.Text, 16) > uiMaxValue)
                txtID.Text = string.Format("{0:X" + iTextLength.ToString() + "}", uiMaxValue);
        }

        private void txtID_KeyPress(object sender, KeyPressEventArgs e)
        {
            char chCheck;

            // We convert the Character to its Upper case equivalent
            //
            chCheck = char.ToUpper(e.KeyChar);

            // The Key is the Delete (Backspace) Key
            //
            if (chCheck == 8)
                return;
            // The Key is a number between 0-9
            //
            if ((chCheck > 47) && (chCheck < 58))
                return;
            // The Key is a character between A-F
            //
            if ((chCheck > 64) && (chCheck < 71))
                return;

            // Is neither a number nor a character between A(a) and F(f)
            //
            e.Handled = true;
        }

        private void txtData0_Leave(object sender, EventArgs e)
        {
            TextBox txtbCurrentTextbox;

            // all the Textbox Data fields are represented with 2 characters.
            // Therefore if the Length of the text is smaller than 2, we add
            // a "0"
            //
            if (sender.GetType().Name == "TextBox")
            {
                txtbCurrentTextbox = (TextBox)sender;
                while (txtbCurrentTextbox.Text.Length != 2)
                    txtbCurrentTextbox.Text = ("0" + txtbCurrentTextbox.Text);
            }
        }
        #endregion

        #region Radio- and Check- Buttons event-handlers


        private void chbExtended_CheckedChanged(object sender, EventArgs e)
        {
            uint uiTemp;

            txtID.MaxLength = (chbExtended.Checked) ? 8 : 3;

            // the only way that the text length can be bigger als MaxLength
            // is when the change is from Extended to Standard message Type.
            // We have to handle this and set an ID not bigger than the Maximum
            // ID value for a Standard Message (0x7FF)
            //
            if (txtID.Text.Length > txtID.MaxLength)
            {
                uiTemp = Convert.ToUInt32(txtID.Text, 16);
                txtID.Text = (uiTemp < 0x7FF) ? string.Format("{0:X3}", uiTemp) : "7FF";
            }

            txtID_Leave(this, new EventArgs());
        }





        #endregion

        #endregion

        #endregion
        private void SendCANMessage(uint canID, byte[] data)
        {
            TPCANHandle m_PcanHandle = PCANBasic.PCAN_USBBUS1;  // Hangi bus üzerinden mesaj göndereceğimizi belirliyoruz.

            TPCANStatus stsResult;
            TPCANMsg message = new TPCANMsg
            {
                ID = canID,                    // Mesajın ID'sini alıyoruz.
                LEN = (byte)data.Length,       // Mesajın veri uzunluğunu belirliyoruz.
                MSGTYPE = TPCANMessageType.PCAN_MESSAGE_STANDARD, // Mesaj tipini belirliyoruz (Standart mesaj)
                DATA = data                     // Veriyi mesajın DATA alanına atıyoruz.
            };

            // Eğer FD mesajı kullanmak istiyorsak, bu durumda CAN FD tipi belirlemeliyiz.
            if (chbFD.Checked)  // Eğer kullanıcı FD mesajı göndermeyi seçmişse
            {
                message.MSGTYPE |= TPCANMessageType.PCAN_MESSAGE_FD;  // Mesajı CAN FD tipiyle işaretliyoruz
            }

            // Mesajı göndermek için PCANBasic.Write fonksiyonunu çağırıyoruz.
            stsResult = PCANBasic.Write(m_PcanHandle, ref message);

            if (stsResult == TPCANStatus.PCAN_ERROR_OK)
            {
                IncludeTextMessage("Message was successfully SENT");
            }
            else
            {
                MessageBox.Show(GetFormatedError(stsResult), "Error Sending CAN Message");
            }
        }
      


        private void radioButton1_0_CheckedChanged_1(object sender, EventArgs e)
        {
            byte[] data = { 0x00, 0x00, 0x02, 0x00, 0x00, 0x00, 0x00, 0x00 };
            SendCANMessage(0x086, data);
            byte[] speed = { 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00 };
            SendCANMessage(0x085, speed);
            if (radioButton1_0.Checked)
            {
                guna2RadialGauge1.Value = 0;
            }
        }

        private void radioButton2_10_CheckedChanged(object sender, EventArgs e)
        {
            byte[] data = { 0x00, 0x00, 0x02, 0x00, 0x00, 0x00, 0x00, 0x00 };
            SendCANMessage(0x086, data);
            byte[] speed = { 0x00, 0x00, 0x10, 0x00, 0x00, 0x00, 0x00, 0x00 };
            SendCANMessage(0x085, speed);
            if (radioButton2_10.Checked)
            {
                guna2RadialGauge1.Value = 10;
            }
        }

        private void radioButton3_20_CheckedChanged(object sender, EventArgs e)
        {
            byte[] data = { 0x00, 0x00, 0x02, 0x00, 0x00, 0x00, 0x00, 0x00 };
            SendCANMessage(0x086, data);
            byte[] speed = { 0x00, 0x00, 0x20, 0x00, 0x00, 0x00, 0x00, 0x00 };
            SendCANMessage(0x086, speed);
            if (radioButton3_20.Checked)
            {
                guna2RadialGauge1.Value = 20;
            }
        }

        private void radioButton4_R_CheckedChanged(object sender, EventArgs e)
        {
            byte[] data = { 0x00, 0x00, 0x03, 0x00, 0x00, 0x00, 0x00, 0x00 };
            SendCANMessage(0x086, data);
            byte[] speed = { 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00 };
            SendCANMessage(0x085, speed);
            if (radioButton4_R.Checked)
            {
                guna2RadialGauge1.Value = 0;
            }
        }
    }
}
