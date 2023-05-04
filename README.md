from netmiko import ConnectHandler
from getpass import getpass 
from netmiko import NetMikoTimeoutException, NetMikoAuthenticationException
from netmiko import Netmiko
from paramiko.ssh_exception import SSHException
from datetime import datetime
import subprocess, re, time, netmiko
import sys
import ipaddress
import socket
import os
import subprocess, re, time, netmiko
import socket
import os
import ipaddress
import smtplib
from email.mime.text import MIMEText
from email.mime.multipart import MIMEMultipart
from email.mime.application import MIMEApplication
from email.header import Header
import traceback
import os
import time
import datetime
from datetime import datetime
import sys
from os import listdir
from os.path import isfile, join
from jinja2 import Environment, FileSystemLoader, select_autoescape
from email.message import EmailMessage

username = sys.argv[1]
password = sys.argv[2]
new_ios_image_filename = sys.argv[3]
remote_http_server = sys.argv[6]
new_ios_size = sys.argv[7]
new_ios_md5 = sys.argv[8]
IP_LIST = sys.argv[5]
Device_Name = sys.argv[9]
todayDate = time.strftime("%d-%m-%y")

def Cisco_ios_image_verify():
    for IP in [IP_LIST]:
        folder1 = IP + '_'+ todayDate
        folder = f'/netdev/job_files/Cisco_ios_upgrade/{folder1}'
        if not os.path.isdir(folder):
            os.makedirs(folder)
        output_file = f'{folder}'
        RTR = {
            'device_type': 'cisco_ios',
            'ip':   IP,
            'username': username,
            'password': password,
        }
        print ('\n Connecting to the Router ' + IP.strip() + '\n')
        try:
            net_connect = ConnectHandler(**RTR)
        except NetMikoTimeoutException:
            print ('Device not reachable - ' + IP )
            continue
        except NetMikoAuthenticationException:
            print ('Authentication Failure - ' + IP )
            continue
        except SSHException:
            print ('Make sure SSH is enabled - ' + IP )
            continue
        cmd_list = [
            "copy http: flash:",
            remote_http_server,
            new_ios_image_filename,
            "\n",
        ]
        output = net_connect.send_command("show flash: | i avail")
        output = re.findall(r"\w+(?= bytes available)", output)
        output = ",".join(output)
        
        check_if_space_available = int(output) - int(new_ios_size)      

        result1 = open(f'{output_file}/ImageCopyLog.txt','a')
        result1.write("\n\n" + "Device IP:" + IP + "\n")
        if int(check_if_space_available) > 5000 :
            now = datetime.now()
            logs_time = now.strftime("%H:%M:%S")
            print(logs_time + ": " + "Sufficient space available on " + IP )
            # f = open("C:\\Users\\aroycho\\Desktop\\logs.txt", "a")
            result1.write("" + "\n" + logs_time + ": " + "Sufficient space available on " + IP + "\n")
            result1.close()
            time.sleep(2)

            start_time = datetime.now()
            print("Copying image to....." + IP + "\n")
            output = net_connect.send_multiline_timing(cmd_list,read_timeout=0)
            print(output) 

            for line in output:
                result1.write(line)
           
            if re.search(r'\sbytes copied\b',output):
                
                now = datetime.now()
                logs_time = now.strftime("%H:%M:%S")
                print("" + logs_time + ": " + "File copied successfully on " + IP)
                
                result1 = open(f'{output_file}/ImageCopyLog.txt','a')
                result1.write = ("" + "\n" + logs_time + ": " + "File copied successfully on " + IP + "\n" )
                result1.close()

                command = "verify /md5 flash:" + new_ios_image_filename
                now = datetime.now()
                logs_time = now.strftime("%H:%M:%S")
                print("Calculating MD5 checksum.....")
                output = net_connect.send_command_timing(command, strip_prompt=False, strip_command=False, read_timeout=0, delay_factor=2)
                print("" + logs_time + ": " + IP + "calculated MD5 is : " + output + "\n" "Expected MD5 is : " + new_ios_md5 )
                try:
                    output = re.search(' = (\w+)',output)
                    print(output)
                
                except AttributeError:
                    output = re.search(' = (\w+)',output), output.group(1)
                
                if new_ios_md5 == str(output.group(1)):
                    
                    now = datetime.now()
                    logs_time = now.strftime("%H:%M:%S")
                    print("" + logs_time + ": " + "MD5 checksum verified on " + IP)
                    
                    result1 = open(f'{output_file}/ImageCopyLog.txt','a')
                    result1.write("" + "\n" + logs_time + ": " + "MD5 checksum verified on " + IP + "\n")
                    result1.close()


                elif new_ios_md5 != str(output.group(1)):
                    now = datetime.now()
                    logs_time = now.strftime("%H:%M:%S")
                    print("" + logs_time + ": " + "MD5 checksum mismatch on " + IP)
                    
                    result1 = open(f'{output_file}/ImageCopyLog.txt','a')
                    result1.write("" + "\n" + logs_time + ": " + "MD5 checksum mismatch on " + IP + "\n" )
                    result1.close()
                    continue

            elif re.search(r'\%Error copying\b' ,output) :
                now = datetime.now()
                logs_time = now.strftime("%H:%M:%S")
                print("" + logs_time + ": " + "Error copying the file to " + IP)
                
                result1 = open(f'{output_file}/ImageCopyLog.txt','a')
                result1.write("" + "\n" + logs_time + ": " + "Error copying the file to " + IP + "\n")
                result1.close()
                continue
            
            elif re.search(r'\%Error opening\b' ,output) :
                now = datetime.now()
                logs_time = now.strftime("%H:%M:%S")
                print("" + logs_time + ": " + "File doesn't exist on " + IP)
                
                result1 = open(f'{output_file}/ImageCopyLog.txt','a')
                result1.write("" + "\n" + logs_time + ": " + "File doesn't exist on " + IP + "\n")
                result1.close()
                continue

            else :
                now = datetime.now()
                logs_time = now.strftime("%H:%M:%S")
                print("" + logs_time + ": " + "File copy didnt complete on " + IP)         
                result1 = open(f'{output_file}/ImageCopyLog.txt','a')
                result1.write("" + "\n" + logs_time + ": " + "File copy didnt complete on " + IP + "\n")
                result1.close()
                continue
    result1.close()

def send_email_transunion():
    for IP in [IP_LIST]:
        folder1 = IP + '_'+ todayDate
        folder = f'/netdev/job_files/Cisco_ios_upgrade/{folder1}'
        if not os.path.isdir(folder):
            os.makedirs(folder)
        output_file = f'{folder}'
        dir = output_file
        if folder1:
            files = [f for f in listdir(dir) if isfile(join(dir, f))]
            print(files)
            subject = f"Cisco ios upgrade - {IP}"
            From = "awx@transunion.com"
            recipients = ["Ashokkumar.L@transunion.com","Arindam.Roychowdhury@transunion.com"]
            recipients1 = ["Ashokkumar.L@transunion.com","Arindam.Roychowdhury@transunion.com"]
            msg =  MIMEMultipart()
            msg["Subject"] = subject
            msg["From"] = From
            msg["To"] = ", ".join(recipients)
            msg["bcc"] = ", ".join(recipients1)

            html = """\
            <html>
            <head></head>
            <body>
                <p>Hi!<br>
                </p>
                <p>Cisco ios upgrades for device attached,please check.<br>
                </p>
                <p>Thanks,<br>
                NEO Operations
                </p>
            </body>
            </html>
            """
            body = MIMEText(html, 'html')  
            msg.attach(body)  # add message body (text or html)

            for f in files:  # add files to the message
                file_path = os.path.join(dir, f)
                attachment = MIMEApplication(open(file_path, "rb").read(), _subtype="txt")
                attachment.add_header('Content-Disposition','attachment', filename=f)
                msg.attach(attachment)

            s = smtplib.SMTP("smtpcorp.transunion.com")
            s.sendmail(msg["From"], recipients, msg.as_string())
            print("Mail sent successfully")
            s.close()

def main():
    # Prompt user for ip address
    # IP address validation and deleting the device from inventory
    try:
        host = socket.gethostbyaddr(IP_LIST)
        if Device_Name == host[0]:
            pass
        else:
            print({'status':False, 'msg':'Failure: {0} doesnt matches the {1}'.format(IP_LIST,host)})
            sys.exit()
    except socket.error as e:
        print({'status':False, 'msg':'Failure: Error: {0}'.format(e)})
        sys.exit()
    try:
        IP = ipaddress.ip_address(IP_LIST)
        Cisco_ios_image_verify()
        send_email_transunion()
    except ValueError:
        print({'status':False, 'msg':'Failure: IP address {0} is not valid'.format(IP_LIST)})
        sys.exit()

if __name__ == "__main__":
    main()
    



from netmiko import ConnectHandler
from getpass import getpass
from netmiko import NetMikoTimeoutException, NetMikoAuthenticationException
from netmiko import Netmiko
from paramiko.ssh_exception import SSHException
from datetime import datetime
import subprocess, re, time, netmiko
import socket
import os
import ipaddress
import smtplib
from email.mime.text import MIMEText
from email.mime.multipart import MIMEMultipart
from email.mime.application import MIMEApplication
from email.header import Header
import traceback
import os
import time
import datetime
from datetime import datetime
import sys
from os import listdir
from os.path import isfile, join
from jinja2 import Environment, FileSystemLoader, select_autoescape
from email.message import EmailMessage

username = sys.argv[1]
password = sys.argv[2]
new_ios_image_filename = sys.argv[3]
reload_wait_time = sys.argv[4]
IP_LIST = sys.argv[5]
Device_Name = sys.argv[6]
todayDate = time.strftime("%d-%m-%y")

def IP_list(IP):
    for IP in [IP_LIST]:
        folder1 = IP + '_'+ todayDate
        folder = f'/netdev/job_files/Cisco_ios_upgrade/{folder1}'
        if not os.path.isdir(folder):
            os.makedirs(folder)
        output_file = f'{folder}'
        RTR = {
            'device_type': 'cisco_ios',
            'ip':   IP,
            'username': username,
            'password': password,
        }
        print ('\n Connecting to the Router ' + IP.strip() + '\n')
        try:
            net_connect = ConnectHandler(**RTR)
        except NetMikoTimeoutException:
            print ('Device not reachable - ' + IP )
            continue
        except NetMikoAuthenticationException:
            print ('Authentication Failure - ' + IP )
            continue
        except SSHException:
            print ('Make sure SSH is enabled - ' + IP )
            continue

        cmd_list = [
            "sh ver",
            "\n",
            "\n",
            "sh ip arp",
            "\n",
            "\n",
            "show interface status | inc connected",
            "\n",
            "\n",
            "show interface status err-disabled",
            "\n",
            "\n",
            "show interface description | inc up",
            "\n",
            "\n",
            "show ip interface brief | inc up",
            "\n",
            "\n",
            "show cdp neighbors | begin Device ID",
            "\n",
            "\n",
            "show etherchannel summary | begin channel-groups",
            "\n",
            "\n",
            "show switch",
            "\n",
            "\n",
            "show environment all",
            "\n",
            "\n",
            "show vlans",
            "\n",
            "\n",
            "sh ip route",
            "\n",
            "\n",
            "sh flash:"
            "\n",
            "\n",
        ]

    start_time = datetime.now()
    for command in cmd_list:
        output = net_connect.send_command(command)
        print(output)
        result1 = open(f'{output_file}/PreCheck.txt','a')
        result1.write(IP + "\n")  
        # for line in output:
        result1.write(output)     
        result1.write("\n" + "\n" + "**************************************************************************************************************************************" + "\n" + "\n")
        result1.close()    
    result2 = open(f'{output_file}/Conf_nd_ReloadLog.txt','a')
    result2.write(IP + "\n")              
    command1 = "no boot system"
    output1 = net_connect.send_config_set(command1)
    print(output1)
    command2 = "boot system flash0:" + new_ios_image_filename
    output2 = net_connect.send_config_set(command2)
    print(output2)
    print('\n Saving the Router configuration \n')
    command3 = "wr memory"
    output3 = net_connect.send_command(command3)
    print(output3)
                        
    for line in output1:
        result2.write(line)
    for line in output2:
        result2.write(line)
    for line in output3:
        result2.write(line)

    now = datetime.now()
    logs_time = now.strftime("%H:%M:%S")
    print("" + logs_time + ": " + "Sending reload command to " + IP)
    confirm_reload = net_connect.send_command('reload', expect_string='[confirm]')
    confirm_reload = net_connect.send_command('\n', expect_string='[confirm]')
    end_time = datetime.now()
    output4 = end_time - start_time
    print(f"\nExec time: {end_time - start_time}\n")

    def sleeptime():
        now = datetime.now()
        logs_time = now.strftime("%H:%M:%S")
        print("" + logs_time + ": Wait time activated for, please wait for " + str(reload_wait_time) + " seconds")
        time.sleep(int(reload_wait_time))        
        result2 = open(f'{output_file}/Conf_nd_ReloadLog.txt','a')
        result2.write = ("" + logs_time + ": " + "Reload command sent to " + IP + "\n")
        result2.close()

    sleeptime()
    result2.write("\n\n\nReload Exec time:" + str(output4))
    print ('\n Connecting to the Router again after reload ' + IP.strip() + '\n')
    for IP in [IP_LIST]:
        folder1 = IP + '_'+ todayDate
        folder = f'/netdev/job_files/Cisco_ios_upgrade/{folder1}'
        if not os.path.isdir(folder):
            os.makedirs(folder)
        output_file = f'{folder}'
        RTR = {
            'device_type': 'cisco_ios',
            'ip':   IP,
            'username': username,
            'password': password,
        }
        try:
            net_connect = ConnectHandler(**RTR)
        except NetMikoTimeoutException:
            print ('Device not reachable after reboot - ' + IP )
            now = datetime.now()
            logs_time = now.strftime("%H:%M:%S")
            result2 = open(f'{output_file}/Conf_nd_ReloadLog.txt','a')
            result2.write = ("" + logs_time + ": " + "Device not reachable after reboot - " + IP + "\n")
            result2.close()
            pass
        except NetMikoAuthenticationException:
            print ('Authentication Failure after reboot - ' + IP)
            now = datetime.now()
            logs_time = now.strftime("%H:%M:%S")
            result2 = open(f'{output_file}/Conf_nd_ReloadLog.txt','a')
            result2.write = ("" + logs_time + ": " + "Authentication Failure after reboot - " + IP + "\n")
            result2.close()
            pass
        except SSHException:
            print ('Make sure SSH is enabled after reboot - ' + IP)
            now = datetime.now()
            logs_time = now.strftime("%H:%M:%S")
            result2 = open(f'{output_file}/Conf_nd_ReloadLog.txt','a')
            result2.write = ("" + logs_time + ": " + "Make sure SSH is enabled after reboot - " + IP + "\n")
            result2.close()
            pass        
        for command in cmd_list:
            output5 = net_connect.send_command(command)
            print(output5)
            print(output5) 
            result3 = open(f'{output_file}/PostCheck.txt','a')
            result3.write(IP + "\n")
            result3.write(output5)
            # for line in output5:
            #     result3.write(line)    
            result3.write("\n" + "\n" + "**************************************************************************************************************************************" + "\n" + "\n")
            result3.close()
            result2.close()
        return IP

def send_email_transunion():
    for IP in [IP_LIST]:
        folder1 = IP + '_'+ todayDate
        folder = f'/netdev/job_files/Cisco_ios_upgrade/{folder1}'
        if not os.path.isdir(folder):
            os.makedirs(folder)
        output_file = f'{folder}'
        dir = output_file
        if folder1:
            files = [f for f in listdir(dir) if isfile(join(dir, f))]
            print(files)
            subject = f"Cisco ios upgrade - {IP}"
            From = "awx@transunion.com"
            recipients = ["Ashokkumar.L@transunion.com","Arindam.Roychowdhury@transunion.com"]
            recipients1 = ["Ashokkumar.L@transunion.com"]
            msg =  MIMEMultipart()
            msg["Subject"] = subject
            msg["From"] = From
            msg["To"] = ", ".join(recipients)
            msg["bcc"] = ", ".join(recipients1)

            html = """\
            <html>
            <head></head>
            <body>
                <p>Hi!<br>
                </p>
                <p>Cisco ios upgrades for device attached,please check.<br>
                </p>
                <p>Thanks,<br>
                NEO Operations
                </p>
            </body>
            </html>
            """
            body = MIMEText(html, 'html')  
            msg.attach(body)  # add message body (text or html)

            for f in files:  # add files to the message
                file_path = os.path.join(dir, f)
                attachment = MIMEApplication(open(file_path, "rb").read(), _subtype="txt")
                attachment.add_header('Content-Disposition','attachment', filename=f)
                msg.attach(attachment)

            s = smtplib.SMTP("smtpcorp.transunion.com")
            s.sendmail(msg["From"], recipients, msg.as_string())
            print("Mail sent successfully")
            s.close()

def main():
    # Prompt user for ip address
    # IP address validation and deleting the device from inventory
    try:
        host = socket.gethostbyaddr(IP_LIST)
        if Device_Name == host[0]:
            pass
        else:
            print({'status':False, 'msg':'Failure: {0} doesnt matches the {1}'.format(IP_LIST,host)})
            sys.exit()
    except socket.error as e:
        print({'status':False, 'msg':'Failure: Error: {0}'.format(e)})
        sys.exit()
    try:
        IP = ipaddress.ip_address(IP_LIST)
        print("{0} address is valid".format(IP))
        IP_list(IP)
        send_email_transunion()
    except ValueError:
        print({'status':False, 'msg':'Failure: IP address {0} is not valid'.format(IP_LIST)})
        sys.exit()

if __name__ == "__main__":
    main()
