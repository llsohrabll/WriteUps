# Appointment

### Nmap

<img width="842" height="179" alt="image" src="https://github.com/user-attachments/assets/3f7f9b3a-f89f-4dc5-8209-e6f5af0a23c7" />

I opened the web page and face this UI:
<img width="1917" height="842" alt="image" src="https://github.com/user-attachments/assets/f45afa51-d50f-4679-8c87-073c4cc32429" />

then i enter username as "admin" and password as "test" then i proxy it to burp and captured the http packet
<img width="1199" height="323" alt="image" src="https://github.com/user-attachments/assets/c72d7f11-1a1f-471d-8d88-3c32ed44af90" />

### sqlmap
i copied the requst and save it as req.txt file then pass it to -r in sqlmap
<img width="1313" height="587" alt="image" src="https://github.com/user-attachments/assets/7582b2a7-b563-4bd2-aca8-050970b01943" />
<img width="893" height="85" alt="image" src="https://github.com/user-attachments/assets/5a998e94-dd59-4c84-b634-278a326f7356" />

Finally i used this sql injection and i clould login as an admin user:
<img width="621" height="392" alt="image" src="https://github.com/user-attachments/assets/3d80b7d7-6f27-4922-956c-1121750a9bcc" />
<img width="359" height="255" alt="image" src="https://github.com/user-attachments/assets/f51a473a-8e50-4275-9501-f3e828acfc75" />


Done !
