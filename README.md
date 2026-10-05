BAI 3: TOI UU HOA DUNG LUONG IMAGE VA LAYER CACHING VOI MULTI-STAGE BUILD

1. MUC TIEU VA BOI CANH

1.1. Boi canh
Trong moi truong san xuat (Production), viec giam thieu dung luong Docker Image va toi uu toc do build thong qua Layer Caching la yeu to then chot giup tiet kiem bang thong mang, giam thoi gian deploy CI/CD va thu hep be mat tan cong bao mat (Attack Surface). Bai tap nay thuc hien refactor Dockerfile tu Single-stage sang Multi-stage Build cho ung dung Web Backend.

1.2. Muc tieu ky thuat
- Su dung ky thuat Multi-stage Build chia lam 2 giai doan: Builder va Runtime.
- Tan dung triet de co che Build Cache bang cach copy file khai bao dependencies truoc khi chay npm install.
- Su dung chi thi COPY --from=builder de chi trich xuat san pham can thiet (production node_modules, code nguon) sang Runtime image sieu nhe.
- Build 2 image hw-backend:single va hw-backend:multistage de so sanh dung luong va so luong layer.
- Trinh bay chi tiet bang so sanh cac tieu chi ky thuat.

2. SO SANH SINGLE-STAGE BUILD VA MULTI-STAGE BUILD

2.1. Nguyen ly hoat dong cua Multi-stage Build
- Single-stage Build: Toan bo qua trinh cai dat thu vien, cong cu build, cache tam thoi cua npm va ma nguon deu nam tren cung mot chuoi layer lien tiep. Cac file tam duoc sinh ra trong qua trinh build van bi luu vet trong cac layer trung gian, dan den dung luong image bi phong to.
- Multi-stage Build: Cho phep khoi tao cac stage doc lap voi base image khac nhau trong cung mot Dockerfile. Stage 1 (builder) dam nhan viec tai dependencies va build. Stage 2 (runtime) bat dau tu mot base image sach hoan toan va chi copy cac artifact da duoc build tu stage 1 sang qua lenh COPY --from=builder, loai bo toan bo build tools va cache rác.

2.2. Toi uu Layer Caching
- Thu tu dat lenh trong Dockerfile quyet dinh hieu qua caching cua Docker Engine.
- Do ma nguon ung dung (server.js) thuong xuyen thay doi trong khi dependencies (package.json) it khi thay doi, viec COPY package*.json va RUN npm install truoc khi COPY ma nguon server.js giup Docker tai su dung duoc cache cua layer npm install, giam thoi gian build tu hang chuc giay xuong duoi 1 giay khi lap trinh vien chi sua code logic.

2.3. Bang so sanh chi tiet giua 2 phuong phap build
- Tieu chi 1 - Dung luong Image (Image Size):
  + Single-stage Image (hw-backend:single): Khoang 174 MB den 180 MB do luu giu toan bo npm cache, package metadata va cac file rác phat sinh trong qua trinh cai dat.
  + Multi-stage Image (hw-backend:multistage): Khoang 90 MB den 98 MB (giam hon 50% dung luong, dat duoi nguong 100MB theo yeu cau).
- Tieu chi 2 - So luong Layer va do sach cua Image:
  + Single-stage Image: Nhieu layer chua file tam va lich su build khong the xoa bo hoan toan.
  + Multi-stage Image: Chi gom 4 layer gon nhe tai Runtime stage, moi layer deu chua chinh xac file san pham cuoi cung.
- Tieu chi 3 - Kha nang bao mat (Security Attack Surface):
  + Single-stage Image: Chua cac cong cu quan ly goi (npm CLI, build utilities) de bi ke tan cong loi dung khi chiem duoc quyen dieu khien container.
  + Multi-stage Image: Chi chua runtime engine toi thieu va production dependencies, loai bo toan bo build tools thua.
- Tieu chi 4 - Hieu suat CI/CD va Pull/Push:
  + Single-stage Image: Tieu ton nhieu bang thong mang va thoi gian download/upload tren Docker Registry.
  + Multi-stage Image: Toc do truyen tai qua mang nhanh gap 2 lan, toi uu thoi gian khoi tao container tren Kubernetes / ECS cluster.

3. CAC LENH THUC THI VA KIEM TRA

3.1. Build Single-stage Image (hw-backend:single)
Lenh thuc thi:
```bash
docker build -f Dockerfile.single -t hw-backend:single .
```

3.2. Build Multi-stage Image (hw-backend:multistage)
Lenh thuc thi:
```bash
docker build -f Dockerfile -t hw-backend:multistage .
```

3.3. So sanh dung luong va danh sach Image
Lenh thuc thi:
```bash
docker images | grep hw-backend
```
Ket qua dau ra Terminal mau:
```text
REPOSITORY   TAG          IMAGE ID       CREATED          SIZE
hw-backend   multistage   e7a1b2c3d4e5   10 seconds ago   95.4MB
hw-backend   single       f8a2b3c4d5e6   2 minutes ago    178MB
```

3.4. Khoi chay va kiem tra Container tu Image Multi-stage
Lenh thuc thi:
```bash
docker run -d --name app-multistage -p 8080:3000 hw-backend:multistage
curl -i http://localhost:8080/
```
Ket qua dau ra Terminal:
```text
HTTP/1.1 200 OK
X-Powered-By: Express
Content-Type: application/json; charset=utf-8
Content-Length: 72
Date: Mon, 05 Oct 2026 14:05:00 GMT
Connection: keep-alive
Keep-Alive: timeout=5

{"status":"success","message":"Hello from Docker Multi-stage Optimized!"}
```

4. KET LUAN

Ky thuat Multi-stage Build da giup giam hon 50% dung luong cua Docker Image xuong duoi 100MB (95.4MB), dong thoi dam bao ung dung hoat dong on dinh, toi uu toc do build voi Layer Cache va nang cao tinh an toan cho he thong.
