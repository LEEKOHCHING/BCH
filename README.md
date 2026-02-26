# TabbyPOS 🐱 | Instant Bitcoin Cash (BCH) Retail Solution

[![BCH-1 Hackcelerator](https://img.shields.io/badge/BCH--1-Hackcelerator-green)](https://bch-1.org)
[![Tech Stack](https://img.shields.io/badge/Stack-VB.NET%20%7C%20SQL-blue)](https://github.com/lkching7/TabbyPOS)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)

**TabbyPOS** is a lightweight, non-custodial Point of Sale (POS) system designed to fulfill the vision of "Peer-to-Peer Electronic Cash." By leveraging **Bitcoin Cash (BCH) 0-conf** technology, TabbyPOS provides a sub-second checkout experience that is faster than traditional credit cards, without compromising merchant security.

---

## 🚀 The Vision

Developed for the **BCH-1 Hackcelerator**, TabbyPOS bridges the gap between digital assets and real-world retail by solving two primary hurdles:
1. **The Confirmation Gap**: Traditional blockchains are too slow for the counter. TabbyPOS uses BCH 0-conf to authorize sales the moment a transaction hits the network.
2. **The Reconciliation Conflict**: In high-volume environments, distinguishing between multiple customers paying the same amount is a nightmare. TabbyPOS solves this with unique **TXID-level tracking**.

---

## 🛠 Core Technical Features



### 1. Instant 0-Conf Authorization
TabbyPOS monitors the **BCH Mempool** (Memory Pool) in real-time. Instead of waiting for a block to be mined (10+ minutes), the system captures the broadcast signal. This allows the merchant to hand over the goods immediately, providing a seamless "Scan and Go" experience.

### 2. SQL-Driven Transaction Reconciliation
To prevent "Accounting Collisions" (where Person A and Person B pay the same amount simultaneously), TabbyPOS implements a robust database-level check:
* **Unique Fingerprinting**: Every incoming payment is indexed by its unique **Transaction ID (TXID)**.
* **Deterministic Accounting**: The system cross-references the live blockchain data with a local SQL index, ensuring that each payment triggers the "Success" event exactly once.

### 3. Merchant Sovereignty (Non-Custodial)
TabbyPOS is designed with freedom in mind. 
* **Direct-to-Wallet**: Funds go straight to the merchant's own address. 
* **No Middleman**: There are no centralized gateways to freeze accounts or charge high commissions. The merchant only pays the sub-cent BCH network fee.

---

## 💻 Technical Implementation

### Advanced Reconciliation Logic (VB.NET)
This core logic demonstrates how TabbyPOS ensures every Satoshi is accounted for using integer precision and TXID validation:

```vb.net
Imports System.Net
Imports System.Data.SqlClient
Imports Newtonsoft.Json.Linq

Public Class TabbyPaymentService
    ' Ensure ConnStr is defined in your configuration or global variables
    Private Const ConnStr As String = "Your_Connection_String_Here"

    <WebMethod()>
    Public Function CheckMempoolForNewPayment(ByVal MerchantAddr As String, ByVal TargetBCH As Double) As Integer
        Try
            ' Force TLS 1.2 for security compliance
            ServicePointManager.SecurityProtocol = CType(3072, SecurityProtocolType)
            
            ' Convert BCH to Satoshis for precise integer comparison: $Sats = \text{Round}(BCH \times 10^8)$
            Dim TargetSats As Long = CLng(Math.Round(TargetBCH * 100000000))

            ' Normalize address for comparison
            Dim myAddr As String = MerchantAddr.ToLower().Trim()
            
            ' Requesting data from local Python Proxy (Port 3030)
            Dim apiUrl As String = "http://127.0.0.1:3030/bch/mempool/" & myAddr

            Using client As New WebClient()
                client.Encoding = System.Text.Encoding.UTF8
                client.Headers.Add("user-agent", "TabbyPOS_Client_2026")

                Try
                    Dim response As String = client.DownloadString(apiUrl)
                    Dim txList As JArray = JArray.Parse(response)

                    ' Iterate through Mempool transactions
                    For Each tx In txList
                        Dim txid As String = tx("txid").ToString()

                        ' Core Deduplication: Check if TXID is already handled in SQL
                        If Not IsTxProcessed(txid) Then

                            ' Aggregate amount from vout (outputs)
                            Dim vouts As JArray = tx("vout")
                            Dim currentTxSats As Long = 0

                            For Each vout In vouts
                                ' Match against scriptpubkey_address field
                                Dim jsonAddr As String = vout("scriptpubkey_address").ToString().ToLower()

                                ' Compare address with or without bitcoincash: prefix
                                If jsonAddr = myAddr OrElse jsonAddr.Replace("bitcoincash:", "") = myAddr.Replace("bitcoincash:", "") Then
                                    currentTxSats += CLng(vout("value"))
                                End If
                            Next

                            ' Final amount verification using Satoshi units
                            If currentTxSats >= TargetSats Then
                                ' Log success to database to prevent duplicate processing
                                SaveProcessedTx(txid, TargetBCH, MerchantAddr)
                                Return 1
                            End If
                        End If
                    Next

                Catch ex As Exception
                    Return 0
                End Try
            End Using

        Catch ex As Exception
            Return 0
        End Try
        Return 0
    End Function

    ''' <summary>
    ''' Check if a Transaction ID (TXID) has already been recorded in the database.
    ''' </summary>
    Private Function IsTxProcessed(ByVal txid As String) As Boolean
        Try
            Using conn As New SqlConnection(ConnStr)
                conn.Open()
                Dim cmd As New SqlCommand("SELECT COUNT(1) FROM BCH_Processed_Payments WHERE TXID = @txid", conn)
                cmd.Parameters.AddWithValue("@txid", txid)
                Return CInt(cmd.ExecuteScalar()) > 0
            End Using
        Catch ex As Exception
            WriteErrror("DB_Check", ex.Message)
            ' Return True on error to fail-safe and prevent accidental duplicate fulfillment
            Return True 
        End Try
    End Function

    ''' <summary>
    ''' Record a successful transaction into the database for reconciliation.
    ''' </summary>
    Private Sub SaveProcessedTx(ByVal txid As String, ByVal amount As Double, ByVal addr As String)
        Try
            Using conn As New SqlConnection(ConnStr)
                conn.Open()
                Dim cmd As New SqlCommand("INSERT INTO BCH_Processed_Payments (TXID, OrderAmount, MerchantAddr, ProcessTime) VALUES (@txid, @amt, @addr, GETDATE())", conn)
                cmd.Parameters.AddWithValue("@txid", txid)
                cmd.Parameters.AddWithValue("@amt", amount)
                cmd.Parameters.AddWithValue("@addr", addr)
                cmd.ExecuteNonQuery()
            End Using
        Catch ex As Exception
            WriteErrror("SaveProcessedTx", ex.Message)
        End Try
    End Sub
End Class
