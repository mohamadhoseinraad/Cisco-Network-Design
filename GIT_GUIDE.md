# Version Control Guide for Cisco Packet Tracer Projects

## Why Use Git for Network Projects?

1. **Track Changes:** See configuration evolution over time
2. **Collaboration:** Share with team members or instructor
3. **Backup:** Never lose your work
4. **Documentation:** README and guides stay with the project
5. **Portfolio:** Showcase your networking skills to employers

## Initial Setup

### 1. Initialize Git Repository
```bash
cd /Users/mohammad/Documents/Code/Cisco-Network-Design
git init
git add .
git commit -m "Initial commit: Complete Cisco network project"
```

### 2. Create GitHub Repository
1. Go to [github.com](https://github.com)
2. Click "New Repository"
3. Name: `Cisco-Network-Design` or `Network-Project-40119783`
4. Description: "Enterprise network with HQ-Branch connectivity using VLANs, GRE tunnel, OSPF, and DHCP"
5. Choose **Public** or **Private**
6. **Don't** initialize with README (you already have one)
7. Click "Create Repository"

### 3. Link Local to GitHub
```bash
# Add remote repository
git remote add origin https://github.com/YOUR_USERNAME/Cisco-Network-Design.git

# Push to GitHub
git branch -M main
git push -u origin main
```

## Best Practices for Network Projects

### What to Commit:
✅ `.pkt` files (Packet Tracer simulations)  
✅ Configuration files (`.txt`)  
✅ Documentation (`.md` files)  
✅ Network diagrams (`.png`, `.pdf`)  
✅ README and guides  
✅ `.gitignore` file  

### What NOT to Commit:
❌ Packet Tracer backup files (`*.bak`)  
❌ Temporary files (`*~`)  
❌ System files (`.DS_Store`, `Thumbs.db`)  
❌ Personal notes with sensitive info  

## Git Workflow for Network Projects

### Making Changes:
```bash
# After modifying .pkt file or configs
git status                          # See what changed
git add 40119783-NetProject.pkt     # Stage .pkt file
git add Configs/R1-HQ_config.txt    # Stage config changes
git commit -m "Updated NAT configuration on R1-HQ"
git push
```

### Commit Message Examples:
```bash
git commit -m "Add: Initial network topology with VLANs"
git commit -m "Fix: DHCP relay configuration on R1-HQ"
git commit -m "Update: OSPF cost on tunnel interface"
git commit -m "Add: ACL for web server security"
git commit -m "Docs: Complete VLAN configuration guide"
```

### Branching Strategy:
```bash
# Create feature branch for major changes
git checkout -b feature/add-ipv6
# Make changes, test, then merge
git checkout main
git merge feature/add-ipv6
git push
```

## Handling Large Files

### If .pkt file is very large (>100MB):
Use Git LFS (Large File Storage):

```bash
# Install Git LFS
brew install git-lfs  # macOS

# Initialize Git LFS
git lfs install

# Track .pkt files
git lfs track "*.pkt"
git add .gitattributes
git commit -m "Add Git LFS for Packet Tracer files"
```

## Repository Organization Tips

### Use Tags for Versions:
```bash
# Tag project milestones
git tag -a v1.0 -m "Complete VLANs and Router-on-Stick"
git tag -a v2.0 -m "Added GRE tunnel and OSPF"
git tag -a v3.0 -m "Final version with ACLs"
git push --tags
```

### Create Releases on GitHub:
1. Go to your repository on GitHub
2. Click "Releases" → "Create a new release"
3. Choose tag: `v3.0`
4. Title: "Final Project Submission"
5. Description: List of features
6. Attach `.pkt` file and PDF documentation
7. Click "Publish release"

## Collaboration Workflow

### If working with a team:
```bash
# Clone repository
git clone https://github.com/USERNAME/Cisco-Network-Design.git

# Before making changes, pull latest
git pull

# Make your changes, then push
git add .
git commit -m "Your changes"
git push
```

### Resolving Conflicts:
```bash
# If someone else pushed changes
git pull
# Fix conflicts in files
git add .
git commit -m "Resolved merge conflict"
git push
```

## Viewing Project History

```bash
# See commit history
git log --oneline --graph

# See specific file history
git log Configs/R1-HQ_config.txt

# Compare versions
git diff HEAD~1 Configs/R1-HQ_config.txt

# Restore previous version
git checkout HEAD~1 Configs/R1-HQ_config.txt
```

## GitHub Repository Enhancements

### Add Topics:
On GitHub, add relevant topics:
- `cisco`
- `packet-tracer`
- `networking`
- `ccna`
- `vlan`
- `ospf`
- `gre-tunnel`

### Add Repository Description:
"Complete enterprise network design featuring VLANs, inter-VLAN routing, DHCP, NAT/PAT, GRE tunneling, OSPF routing, and ACL security - implemented in Cisco Packet Tracer"

### Create GitHub Wiki (Optional):
Add extra documentation:
1. Go to repository → Wiki
2. Create pages for:
   - Troubleshooting FAQ
   - Lab exercises
   - Additional resources

## Automated Documentation

### Use GitHub Actions (Advanced):
Create `.github/workflows/validate.yml`:
```yaml
name: Validate Configs
on: [push]
jobs:
  validate:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v2
      - name: Check config files
        run: |
          echo "Validating configuration files..."
          grep -r "ip address" Configs/ || exit 1
```

## Backup Strategy

### Multiple Backups:
1. **GitHub** (primary)
2. **Local Git** (secondary)
3. **Cloud Storage** (tertiary - Google Drive, Dropbox)

```bash
# Backup to external drive
cp -R /Users/mohammad/Documents/Code/Cisco-Network-Design /Volumes/Backup/
```

## Portfolio Presentation

### README Badges:
Add to top of README.md:
```markdown
![Cisco Packet Tracer](https://img.shields.io/badge/Cisco-Packet_Tracer-1BA0D7?logo=cisco)
![Status](https://img.shields.io/badge/Status-Completed-success)
```

### GitHub Profile:
Pin this repository on your GitHub profile to showcase to employers.

## Submission for University

### Export for Submission:
```bash
# Create submission archive
cd /Users/mohammad/Documents/Code
zip -r 40119783-Network-Project.zip Cisco-Network-Design/ \
  -x "*.git*" -x "*.DS_Store"
```

### Include in Submission:
1. ✅ `.pkt` file
2. ✅ All configuration files
3. ✅ README.md
4. ✅ Documentation folder
5. ✅ Project PDF report
6. ✅ GitHub repository link

## Useful Git Commands Reference

```bash
# Check status
git status

# Add all changes
git add .

# Commit with message
git commit -m "Your message"

# Push to GitHub
git push

# Pull latest changes
git pull

# View commit history
git log --oneline

# Undo last commit (keep changes)
git reset --soft HEAD~1

# Undo last commit (discard changes)
git reset --hard HEAD~1

# Create branch
git checkout -b branch-name

# Switch branch
git checkout main

# Merge branch
git merge branch-name

# Clone repository
git clone https://github.com/USER/REPO.git
```

## Summary

Your GitHub repository structure:
```
yourusername/Cisco-Network-Design
├── Clean README.md with badges
├── Organized folder structure
├── Complete documentation
├── Version-controlled configs
├── Tagged releases
└── Professional presentation
```

This makes your project:
- **Shareable** with instructors/peers
- **Portfolio-ready** for job applications
- **Well-documented** for future reference
- **Backed up** and version-controlled

---

**Next Step:** Push your complete project to GitHub and share the link! 🚀
