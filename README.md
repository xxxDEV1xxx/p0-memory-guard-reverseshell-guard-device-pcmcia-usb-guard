Copyright (c) 2026 Christopher T. Williams

License

Permission is granted to use, copy, modify, and distribute this software solely for non-commercial research, testing, and security-analysis purposes, provided that the copyright notice and this permission notice are retained.

Commercial use, sale, marketplace distribution, or monetization of this software, in whole or in part, is strictly prohibited without prior written consent from the author.

THIS SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY, FITNESS FOR A PARTICULAR PURPOSE, AND NON-INFRINGEMENT. THE AUTHOR SHALL NOT BE LIABLE FOR ANY CLAIM, DAMAGES, OR OTHER LIABILITY ARISING FROM THE USE OR OTHER DEALINGS IN THE SOFTWARE.


Verify sha
Linux: sha256sum *.tar.gz
Windows: Get-FileHash P0-reverseshell-v3-production-.tar.gz -algorithm sha256

P0-reverseshell-v3-production-.tar.gz
73f20299702b768461859cfc7f7e71db4a5fbdafbeda23cfb7af1d1b55776801 

P0-memory-identity-beastmode-.tar.gz
32f6bcef292da63cf557fe6d47e71d3e0edd946f83379da5a44a8748397b7e92 

pcmciaguard-.tar.gz
5a6b7c796993494432ccb786f67e2fb066b35a8a4fc16b399632a399209a1864  

p0-firewall-recreation.tar.gz
170a0e48451abde18694180c18f7c94e5237d9afc350d92f064ed6feb58c0f15

P0-transform-clean.tar
f60df08767cab33469b73a749ec250ff56b8294b6729fc9836c91668683f65f3  


"If you make an iso of your OS,  burn it to CD.
If you burn it to CD, dd if=/dev/sd0 of=/dev/sda1 conv=noerror,sync status=progress and burn it to a drive.
If you burn it to a drive, add another partition on the rest of the space.
If you add another partition, sda1 is iso9660 write proofed, so bindmount and overlay folders on sda2
The boot OS is now nearly immutable with few exceptions."
Christopher T. Williams
