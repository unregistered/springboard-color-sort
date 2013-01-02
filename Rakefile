#
# Things you should edit
#  
USER = 'mobile'
HOST = 'Chris-Lis-iPhone.local'
WORKSPACEDIR = "_tmp"
SKIP_FIRST_N_PAGES = 1 # How many pages of icons to not touch. If set to 1, 
                       # the first page won't be included in sorting. 
ICONS_PER_PAGE = 16 # Assuming iPhone < 5


#
# Things you shouldn't have to edit
#
require 'pry' if ENV['DEBUG']
require 'pathname'
require 'plist'
require 'mini_magick'
module MiniMagick
  class Image
    def pixel_at(x, y)
      case run_command("convert", "#{escaped_path}[1x1+#{x}+#{y}]", "-depth 8", "txt:").split("\n")[1]
      when /^0,0:.*(#[\da-fA-F]{6}).*$/ then $1
      else nil
      end
    end
  end
end

class Array
  def springboard_flatten
    new_a = []
    self.flatten.each do |item|
      if item.is_a? String
        # Bare icon, insert it directly
        new_a.push item
      elsif item.is_a? Hash
        # Means we have a folder
        item['iconLists'].flatten.each do |bundleid|
          new_a.push bundleid
        end
      else
        raise "Found non-string or non-hash in IconState.plist. Please open an issue and upload your IconState.plist file."
      end
    end
    
    return new_a
  end
end

def remote_sh cmd
  sh "ssh #{USER}@#{HOST} '#{cmd}'"
end

class App  
  def initialize folder
    @folder = folder
  end
  
  def bundle_id
    return @_bundle_id if @_bundle_id
    
    # Lookup the bundle id
    data = Plist::parse_xml self.info_plist
    @_bundle_id = data['CFBundleIdentifier']
  end
  
  def app_folder
    return @_app_folder if @_app_folder
    
    @_app_folder = @folder.children.find{|c| c.basename.to_s =~ /.app/}
  end
  
  def info_plist
    self.app_folder + "Info.plist"
  end
  
  def image_file
    candidate = self.app_folder + "iTunesArtwork"
    return candidate if candidate.file?
    
    candidate = @folder + "iTunesArtwork"
    return candidate if candidate.file?
    
    raise IOError.new("File not found")
  end
  
  def dominant_color
    return @_color if @_color
    
    img = MiniMagick::Image.open(self.image_file)
    img.resize "1x1"
    hex = img.pixel_at(0, 0)
    rgb = [hex[1..2].to_i(16), hex[3..4].to_i(16), hex[5..6].to_i(16)]
    hsv = rgb_to_hsv(rgb)
  end
  
  private
  def rgb_to_hsv rgb
    # Taken from http://ntlk.net/2011/11/21/convert-rgb-to-hsb-hsv-in-ruby/
    r = rgb[0] / 255.0
    g = rgb[1] / 255.0
    b = rgb[2] / 255.0
    max = [r, g, b].max
    min = [r, g, b].min
    delta = max - min
    v = max * 100
 
    if (max != 0.0)
    	s = delta / max *100
    else
    	s = 0.0
    end
 
    if (s == 0.0) 
    	h = 0.0
    else
    	if (r == max)
    		h = (g - b) / delta
    	elsif (g == max)
    		h = 2 + (b - r) / delta
    	elsif (b == max)
    		h = 4 + (r - g) / delta
    	end
 
    	h *= 60.0
	
    	if (h < 0)
    		h += 360.0
    	end
    end
    
    return [h, s, v] # h => 0..360, s => 0...100, v => 0...100
  end
end

task :default => [:clean, :setup, :fetch, :sort, :push, :respring]

desc "Displays help"
task :help do
  sh "rake -T"
end

desc "Teardown folders"
task :clean do
  rm_rf WORKSPACEDIR
end

desc "Create working dir"
task :setup do
  mkdir_p WORKSPACEDIR
end

desc "Brings resources down"
task :fetch do
  puts "\n\nBring plist down"
  sh "scp #{USER}@#{HOST}:/User/Library/SpringBoard/IconState.plist ./#{WORKSPACEDIR}"
  sh "cp ./#{WORKSPACEDIR}/IconState.plist ./#{WORKSPACEDIR}/IconState.plist.orig"
  sh "plutil -convert xml1 ./#{WORKSPACEDIR}/IconState.plist"
  
  puts "\n\nBring icons and metadata down (this will take some time)"
  remote_sh "cd /User && find ./Applications/ -type f \\( -name \"iTunesArtwork\" -o -wholename \"*.app/Info.plist\" \\) -exec tar uf data.tar '{}' +"
  sh "scp #{USER}@#{HOST}:/User/data.tar #{WORKSPACEDIR}/"
  sh "cd #{WORKSPACEDIR} && tar -xf data.tar"
  sh "cd #{WORKSPACEDIR}/Applications && plutil -convert xml1 */*/Info.plist"
  remote_sh "rm /User/data.tar"
end

desc "Sort by color and overwrite IconState.plist locally"
task :sort do
  puts "\n\nLoading applications.."
  base = Pathname.new "#{WORKSPACEDIR}/Applications"
  apps = base.children.map do |folder|
    App.new folder if folder.directory?
  end
  
  puts "Analyzing colors.."
  lut = {} # Color lookup table
  apps.each do |app|
    next unless app
    lut[app.bundle_id] = app.dominant_color[0]
  end

  puts "Loading SpringBoard App layout..."
  iconstate = Pathname.new "#{WORKSPACEDIR}/IconState.plist"
  icondata = Plist::parse_xml iconstate
  combined_lut = {} # All icons merged with colors
  icondata['iconLists'].springboard_flatten.each do |bundleid|
    combined_lut[bundleid] = lut[bundleid] || -1
    puts "Icon not found for #{bundleid}" unless lut[bundleid]
  end
  
  puts "Sorting..."
  sorted = combined_lut.sort_by{|k,v| v}
  
  # Emit plist  
  # iconLists is an array of arrays
  newicondata = icondata.dup
  newicondata['iconLists'] = []
  
  # Don't touch the first N pages
  (0...SKIP_FIRST_N_PAGES).each do |idx|
    page = icondata['iconLists'][idx]
    newicondata['iconLists'].push page
    sorted.reject! do |item|
      true if page.springboard_flatten.include? item[0]
    end
  end
  
  
  # Lay down the unsorted elements
  unsortable = sorted.select{|s| s[1] < 0}
  while unsortable.size > 0
    # Make a page
    page = []
    (0...ICONS_PER_PAGE).each do |idx|
      app = unsortable.shift
      page.push app[0] if app
    end
    
    newicondata['iconLists'].push page
  end

  # Lay down the sorted elements
  sortable = sorted.select{|s| s[1] >= 0}
  while sortable.size > 0
    # Make a page
    page = []
    (0...ICONS_PER_PAGE).each do |idx|
      app = sortable.shift # Remove from sorted list
      page.push app[0] if app # Add to page. app is nil if we run out of icons
    end
    newicondata['iconLists'].push page
  end
  
  # Last sanity check before we output the file
  old_icon_count = icondata['iconLists'].springboard_flatten.count
  new_icon_count = newicondata['iconLists'].springboard_flatten.count
  raise "Inconsistency error: #{old_icon_count} != #{new_icon_count}" if old_icon_count != new_icon_count
  
  ofile = Pathname.new "#{WORKSPACEDIR}/IconStateToBePushed.plist"
  ofile.open("w") do |io|
    io << newicondata.to_plist
  end
  
end

desc "Upload Iconstate.plist to the iphone"
task :push do
  sh "plutil -convert binary1 #{WORKSPACEDIR}/IconStateToBePushed.plist"
  remote_sh "cp /User/Library/SpringBoard/IconState.plist /User/Library/SpringBoard/IconState.plist.bak"
  sh "scp #{WORKSPACEDIR}/IconStateToBePushed.plist #{USER}@#{HOST}:/User/Library/SpringBoard/IconState.plist"
end

desc "Respring the device"
task :respring do
  remote_sh "killall SpringBoard"
end